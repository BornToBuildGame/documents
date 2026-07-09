# TDD-06: Tournaments

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Technical Design:** Tournaments  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Technical Architecture

---

## 1. Purpose & Scope

Define the requirements for a tournament system that supports scheduled competitive events with entry limits, durations, multiple seasons, and rewards. Tournaments build on the leaderboard system to provide time-bounded competitive experiences.

---

Refer to [BRD-06](../BRD/06_tournaments.md) for the business requirements and [PRD-06](../PRD/06_tournaments.md) for the API surface.

---

## 2. Architecture & Design Flow

The tournament engine uses a background scheduler thread to manage open, active, and expired tournament phases according to Cron schedules. Leaderboard records store player entries, scoped to each tournament occurrences' boundary.

### Tournament Lifecycle and Reward Flow
```mermaid
sequenceDiagram
    autonumber
    participant Engine as Scheduler Engine
    actor Client as Client/Player
    participant DB as PostgreSQL DB
    participant Hook as Runtime Hook (Rewards)

    loop Every 30 seconds
        Engine->>DB: Query scheduled tournaments
        Engine->>Engine: Evaluate active occurrences via cron
        Engine->>DB: Update 'start_time' timestamps
    end

    Client->>DB: Submit score (verifies remaining attempts & bounds)
    DB-->>Client: Return placement update

    alt Occurrence Expires
        Engine->>Engine: Detect end of occurrence
        Engine->>DB: Fetch top placements
        DB-->>Engine: Top players list
        Engine->>Hook: Trigger tournament_end(tournament, placements)
        Hook->>DB: Grant wallet rewards or items (atomic transaction)
        DB-->>Hook: Confirmation
    end
```

---

## 3. Database Schema & Data Models

Tournaments do not use a standalone database table. Instead, they leverage the `leaderboard` table (see [TDD-05](./05_leaderboards.md)) for metadata and scheduling rules (`duration`, `start_time`, `end_time`, `join_required`, and `max_size`).

### Backing Records and Tournament Joins
- **Participation State**: A player's tournament join registration is tracked directly in `leaderboard_record` (see [TDD-05](./05_leaderboards.md)).
- **Join Operation**: When a player joins a tournament, the engine executes an idempotent insert into `leaderboard_record` with `num_score = 0`, `score = 0`, and `max_num_score` populated.
  ```sql
  INSERT INTO leaderboard_record (leaderboard_id, owner_id, expiry_time, username, num_score, max_num_score)
  VALUES ($1, $2, $3, $4, 0, $5)
  ON CONFLICT (owner_id, leaderboard_id, expiry_time) DO NOTHING
  ```
- If `max_size` is defined on the tournament, joining additionally updates the size count inside `leaderboard`:
  ```sql
  UPDATE leaderboard SET size = size + 1 WHERE id = $1 AND size < max_size
  ```

---

## 4. Algorithmic Logic & Execution Flow

### Active Occurrence Scheduling Algorithm
1. Parse the `reset_schedule` cron pattern (e.g., `0 0 * * 1` for weekly on Mondays).
2. Given $T_{current} = \text{now}$:
   - Find the previous cron trigger time $T_{prev}$ and the next scheduled trigger $T_{next}$.
   - If $T_{current} \ge T_{prev}$ and $T_{current} < T_{prev} + \text{duration}$:
     - The occurrence is active.
     - Set $T_{expiry} = T_{prev} + \text{duration}$.
   - If $T_{current} \ge T_{prev} + \text{duration}$:
     - The occurrence is completed/inactive.
3. Partition player records between occurrences by scoping all query searches and inserts by the calculated occurrence's `expiry_time`.

### Tournament Join & Score Submission Logic
1. **Join Validation**:
   - Check if tournament is active (current time falls between $T_{prev}$ and $T_{prev} + \text{duration}$).
   - Insert idempotent participant record (`num_score = 0`). If a row is affected, increment `size` in `leaderboard` up to `max_size`.
2. **Score Submission Validation**:
   - Verify tournament exists and is active.
   - If `join_required` is enabled:
     - Check if a record for the player exists in `leaderboard_record` for the current occurrence's `expiry_time`. If no record is found, reject the score submission.
     - If a record is found, update the score following the operators (best, set, increment, decrement), increment `num_score` by 1, and update timestamps.

### Go Tournament End Hook Example

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
)

func OnTournamentEnd(ctx context.Context, logger interface{}, db *sql.DB, nk interface{}, tournamentID string, endActive int64, resetActive int64) error {
	// Fetch top 3 players from backing leaderboard
	// Simulated leaderboard record fetch for representation:
	records := []struct {
		OwnerID  string
		Username string
	}{
		{OwnerID: "user-uuid-1", Username: "shadow_ninja"},
		{OwnerID: "user-uuid-2", Username: "super_player"},
		{OwnerID: "user-uuid-3", Username: "racer_x"},
	}

	rewards := []map[string]int64{
		{"coins": 5000, "gems": 50}, // 1st place
		{"coins": 2500, "gems": 20}, // 2nd place
		{"coins": 1000, "gems": 5},  // 3rd place
	}

	for index, record := range records {
		if index >= len(rewards) {
			break
		}
		reward := rewards[index]
		if record.OwnerID != "" {
			// nk.WalletUpdate(ctx, record.OwnerID, reward, metadata)
			fmt.Printf("Rewarded player %s for rank %d\n", record.Username, index+1)
		}
	}

	return nil
}
```

---

## 5. Performance & Security Considerations

### Performance
- **Scheduler Efficiency**: The 30-second scheduler loop queries all tournaments. Use a **next-fire-time priority queue** to avoid scanning completed/inactive tournaments.
- **Reward Distribution**: Distribute rewards in batched transactions (50 players per batch) to avoid long-held database locks during large tournament endings.
- **Leaderboard Record Expiry**: Ensure `expiry_time` is indexed and expired records are pruned within 1 hour of occurrence end to reclaim storage.
- **Concurrent Tournaments**: Support up to **100 active tournaments simultaneously** per server node.

### Security
- **Reward Idempotency**: Track occurrence-level rewards (e.g. `rewarded_at` timestamps or occurrence flags) in-memory or in a distributed Redis key rather than PostgreSQL. Before distributing rewards, evaluate the flag to prevent duplicate reward distribution on scheduler restarts.
- **Authoritative Score Submission**: When `authoritative = TRUE`, only server-side match handlers can submit tournament scores. Client-submitted scores must be rejected.
- **Max Score Attempts**: Enforce `max_num_score` strictly. Track submission count per player per occurrence and reject excess submissions with `RESOURCE_EXHAUSTED`.
- **Input Validation**:
  - `duration`: Must be >0 and ≤604,800 (7 days in seconds).
  - `reset_schedule`: Validate cron expression syntax before persisting. Reject invalid patterns.
  - `max_size`: If >0, enforce capacity check atomically during score submission.

---

## 6. Linked Documents
- [BRD-06](../BRD/06_tournaments.md) (Business Requirements Document)
- [PRD-06](../PRD/06_tournaments.md) (Product Requirements Document)
