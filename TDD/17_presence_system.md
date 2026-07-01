# TDD-17: Presence System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Technical Design:** Presence System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Technical Architecture

---

## 1. Purpose & Scope

Define the requirements for a real-time presence system that tracks the online status, current activity, match participation, party status, and other live state information of connected users. Presence is foundational to social features, matchmaking, and real-time coordination.

---

Refer to [BRD-17](../BRD/17_presence_system.md) for the business requirements and [PRD-17](../PRD/17_presence_system.md) for the API surface.

---

## 2. Architecture & Design Flow

Presence tracking operates as an in-memory subscription engine. Active connection states are tracked using Stream modes. In a clustered deployment, status changes are fanned out across nodes using Redis Pub/Sub or a clustered Gossip protocol.

### Status Subscription & Status Update Flow
```mermaid
sequenceDiagram
    autonumber
    actor A as Player A
    participant Server as Server Core
    participant Cache as Redis PubSub (Cluster)
    actor B as Subscriber (Player B)

    B->>Server: (WS) status_follow {user_ids: [A]}
    Server->>Server: Register B's session as subscriber of A

    A->>Server: (WS) status_update {status: "in_match"}
    Server->>Server: Update A's presence state map
    Server->>Cache: Publish status_change message for user A
    
    Cache-->>Server: Route message to nodes hosting subscribers
    Server->>Server: Identify B is subscriber of A
    Server-->>B: (WS) Pushes 'status_presence_event' {joins: [A: "in_match"]}
```

---

## 3. Database Schema & Data Models

Real-time presence is highly volatile and is held entirely in-memory to prevent disk write bottlenecks. No persistent PostgreSQL schemas are utilized.

### Go Presence Data Structures

```go
package main

import "time"

type Stream struct {
	Mode       int    `json:"mode"`       // 0=status, 1=custom, 2=chat, 3=party, 4=match
	Subject    string `json:"subject"`    // Unique identifier (e.g. matchId, partyId, channelId)
	Subcontext string `json:"subcontext"` // Additional subcategory identifier
	Label      string `json:"label"`      // Metadata descriptor
}

type PresenceRecord struct {
	UserID    string    `json:"user_id"`
	SessionID string    `json:"session_id"`
	Username  string    `json:"username"`
	Node      string    `json:"node"`
	Status    string    `json:"status"` // Current JSON status string
	JoinedAt  time.Time `json:"joined_at"`
}

// In-Memory Registry Representations
type StreamPresences map[string][]PresenceRecord // Key: string representation of Stream
type UserSubscriptions map[string][]string      // Key: watched userId, Value: slice of subscriber sessionIds
```

---

## 4. Algorithmic Logic & Execution Flow

### Stream-based Observer Fan-Out Algorithm
1. When a client performs a `status_follow(target_ids)`:
   - For each target ID, insert the client's `session_id` into the target's `UserSubscriptions` observer set.
   - Look up if the target is currently online: check active presence registry. If online, immediately send a status join event back to the subscriber.
2. When a client performs `status_update(status_payload)`:
   - Update the client's active status value in the memory registry.
   - Fetch the list of subscriber session IDs from `UserSubscriptions` for the client.
   - If in a multi-node cluster, publish the update to the cluster-wide Pub/Sub channel.
   - Loop through active local sessions and write the WebSocket presence frame.

### Go Custom Stream Join & Send Example

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
)

func JoinCustomStream(ctx context.Context, nk interface{}, stream Stream, userID string, sessionID string) error {
	// Join the specified real-time stream via the Nakama framework
	// success, err := nk.StreamUserJoin(stream.Mode, stream.Subject, stream.Subcontext, stream.Label, userID, sessionID, false, false, "")
	success := true
	if !success {
		return errors.New("STREAM_JOIN_FAILED")
	}

	eventPayload := map[string]string{
		"event":   "user_joined",
		"user_id": userID,
	}
	payloadBytes, err := json.Marshal(eventPayload)
	if err != nil {
		return err
	}

	// nk.StreamSend(stream.Mode, stream.Subject, stream.Subcontext, stream.Label, string(payloadBytes), nil, false)
	_ = payloadBytes
	return nil
}
```

---

## 5. Linked Documents
- [BRD-17](../BRD/17_presence_system.md) (Business Requirements Document)
- [PRD-17](../PRD/17_presence_system.md) (Product Requirements Document)
