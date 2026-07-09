# TDD-02: Multiplayer Matchmaking

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Technical Design:** Multiplayer Matchmaking  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Technical Architecture

---

## 1. Purpose & Scope

Define the technical design for a flexible matchmaking system that pairs players into compatible game sessions based on skill, region, game mode, and custom properties. The system supports solo players, parties, and custom matchmaking logic.

---

Refer to [BRD-02](../BRD/02_multiplayer_matchmaking.md) for the business requirements and [PRD-02](../PRD/02_multiplayer_matchmaking.md) for the API surface.

---

## 2. Architecture & Design Flow

The matchmaking engine processes queued tickets asynchronously using an interval loop (ticks). When matchmaking forms a compatible match, a WebSocket payload notifies players with a one-time connection token.

### Matchmaking System Flow
```mermaid
sequenceDiagram
    autonumber
    actor Client A
    actor Client B
    participant Server as Server Core Node
    participant Cache as Distributed Redis Mesh

    Client A->>Server: (WebSocket) Submit ticket to 'ranked_pvp'
    Server->>Cache: Add Client A ticket with properties
    Client B->>Server: (WebSocket) Submit ticket to 'ranked_pvp'
    Server->>Cache: Add Client B ticket with properties

    loop Matchmaker Loop Tick (e.g. every 1 second)
        Server->>Cache: Fetch active tickets for 'ranked_pvp' across all nodes
        Server->>Server: Run pairing algorithm (expand skill window per wait time)
        Note over Server: Match Found between Client A and Client B!
    end

    Server->>Cache: Delete processed tickets from cache
    Server->>Cache: Register matched pair and assign to an Authoritative Node
    Server-->>Client A: (WebSocket) Emit 'matchmaker_matched' with match ID & token
    Server-->>Client B: (WebSocket) Emit 'matchmaker_matched' with match ID & token
```

---

## 3. Distributed In-Memory State (Redis)

Because matchmaking tickets and active matches are ephemeral and highly volatile, they are no longer stored in PostgreSQL to avoid MVCC bloat and write contention.

### Matchmaking Tickets (Redis Sorted Set & Hash)
- **Ticket Index (Sorted Set):** `matchmaker:queue:{queue_name}` (Score: `created_at` epoch).
- **Ticket Payload (Hash):** `matchmaker:ticket:{ticket_id}` containing `user_id`, `skill_rating`, `region`, `properties`, and `party_members`.

### Active Matches (Redis Hash)
- **Match Registry:** `matchmaker:match:{match_id}` containing `players`, `match_token`, `region`, and the `node_id` assigned to host the match.

---

## 4. Algorithmic Logic & Execution Flow

### Matchmaking Matching & Window Expansion Algorithm
1. The matchmaking engine ticks every `1000ms`. In a multi-node cluster, a distributed lock (e.g., Redlock) ensures only one node evaluates a specific queue per tick to avoid race conditions.
2. For each queue (e.g., `ranked_pvp`):
   - Query all active tickets from the Redis Sorted Set `matchmaker:queue:{queue_name}`.
   - Sort tickets by `created_at` in ascending order (longest waiting first).
   - For each ticket $T$:
     - Calculate elapsed wait time: $W = \text{now} - T.\text{created\_at}$.
     - Determine the allowed skill delta range based on $W$ (based on the progression curve).
     - Filter candidate tickets whose skill ratings fall within $[T.\text{skill} - \delta, T.\text{skill} + \delta]$ and match $T.\text{region}$.
     - If `reverse_precision` is active, verify bidirectional compatibility: ensure $T$'s properties also satisfy each candidate ticket's parameters.
     - Group matching candidates up to the queue's `max_count`. If candidates reach `min_count`, form a match.

### Operational Topology Execution

The matchmaking process executes differently depending on the deployment profile:
- **Single-Node Mode:**
  - **Memory Localization:** Matchmaking tickets are placed directly into thread-safe in-memory slices. No network roundtrips to Redis are required.
  - **Local Loop Ticker:** A local background Go goroutine ticks every `1000ms`, evaluating the local queue slices directly. Redlock and distributed lock overhead is bypassed.
- **Multi-Node Mode:**
  - **Redis Serialization:** Tickets are serialized and saved via `HSET` to `matchmaker:ticket:{ticket_id}` and indexed by `ZADD` in the sorted set `matchmaker:queue:{queue_name}`.
  - **Redlock Coordination:** To prevent duplicate match forming or race conditions where Node A and Node B attempt to process the same tickets simultaneously, nodes must acquire a Redis-backed Redlock (`lock:matchmaker:{queue_name}`) before executing the tick algorithm.
  - **Match Spawning:** Once a match is formed, the coordinator node registers the match mapping `matchmaker:match:{match_id}` in Redis and assigns it to a targeted server node based on resource load.

### Skill Range Expansion Config
```
Wait Time (seconds) | Skill Delta Range Allowed (±)
------------------- | ----------------------------
0-5 seconds         | ±50 MMR
5-10 seconds        | ±75 MMR
10-20 seconds       | ±125 MMR
20-30 seconds       | ±200 MMR
30-60 seconds       | ±350 MMR
>60 seconds         | ±500 MMR (Maximum cap)
```

### Go Matching Evaluator Example

```go
package main

import (
	"math"
	"time"
)

type Ticket struct {
	TicketID    string
	UserID      string
	SkillRating float64
	CreatedAt   time.Time
	Region      string
}

func GetSkillDelta(waitSeconds float64) float64 {
	if waitSeconds < 5 {
		return 50
	}
	if waitSeconds < 10 {
		return 75
	}
	if waitSeconds < 20 {
		return 125
	}
	if waitSeconds < 30 {
		return 200
	}
	if waitSeconds < 60 {
		return 350
	}
	return 500
}

func EvaluatePairing(t1, t2 Ticket) bool {
	if t1.Region != t2.Region {
		return false
	}

	now := time.Now()
	wait1 := now.Sub(t1.CreatedAt).Seconds()
	wait2 := now.Sub(t2.CreatedAt).Seconds()

	delta1 := GetSkillDelta(wait1)
	delta2 := GetSkillDelta(wait2)

	diff := math.Abs(t1.SkillRating - t2.SkillRating)

	// Both tickets must satisfy each other's skill range constraints
	return diff <= delta1 && diff <= delta2
}
```

---

## 6. Performance & Security Considerations

### Performance
- **Tick Loop Scalability**: The matchmaker evaluates all queued tickets per tick. At >5,000 concurrent tickets, apply **bucket-based skill grouping** (partition tickets into ±100 MMR bands) to reduce pairing evaluation from O(N²) to O(N × B) where B is bucket size.
- **Maximum Ticket Pool**: Cap at **10,000 active tickets per queue**. Beyond this threshold, reject new submissions with `RESOURCE_EXHAUSTED` and alert operators.
- **Ticket TTL**: Enforce `expires_at` strictly. Expired tickets must be garbage-collected every tick cycle, not left in the queue.
- **Memory Budget**: Each ticket should consume ≤2 KB in-memory. With 10,000 tickets, total matchmaker memory ≤20 MB per queue.
- **Latency Target**: Each matchmaker tick must complete within **500ms** (p99). If tick processing exceeds this, split queues across dedicated goroutines.

### Security
- **Rate Limiting**: Max **1 active matchmaking ticket per user** at any time. Reject duplicate submissions with `ALREADY_EXISTS`.
- **Input Validation**:
  - `skill_rating`: Must be within `[0, 10000]` range. Reject outliers.
  - `queue_name`: Max 64 characters, alphanumeric and underscore only.
  - `properties` JSONB: Max 4 KB, max nesting depth of 3.
  - `min_count` / `max_count`: Must satisfy `2 ≤ min_count ≤ max_count ≤ 100`.
- **Abuse Prevention**: Track ticket submission frequency per user. If a user submits and cancels >10 tickets within 5 minutes, impose a 5-minute matchmaking cooldown.
- **Match Token Security**: The `match_token` issued on match formation must be cryptographically random (≥128 bits), single-use, and expire within 30 seconds.

---

## 5. Linked Documents
- [BRD-02](../BRD/02_multiplayer_matchmaking.md) (Business Requirements Document)
- [PRD-02](../PRD/02_multiplayer_matchmaking.md) (Product Requirements Document)
