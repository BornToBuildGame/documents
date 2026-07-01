# PRD-15: Match State & Persistence

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Match State & Persistence  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for persisting match state, match history, replay metadata, and match statistics. This system enables game state saving/loading, match replay, post-match analysis, and historical statistics tracking.

---

Refer to [BRD-15](../BRD/15_match_state_persistence.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Match history queries are accessible externally via REST and gRPC. Writing match history, creating checkpoints, and reloading states are restricted to server-side code executing on the authoritative runtime.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. List Player Match History
Retrieves a list of completed matches involving a specific player.
* **Endpoint**: `GET /v2/match/history`
* **Query Parameters**:
  * `user_id` (string, optional, default: current user's ID): The user to query match history for.
  * `limit` (integer, optional, default: `20`, max: `100`)
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**:
  ```json
  {
    "matches": [
      {
        "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
        "match_type": "battle_royale",
        "winner_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "duration_sec": 420,
        "start_time": "2026-07-01T22:00:00Z",
        "end_time": "2026-07-01T22:07:00Z",
        "metadata": {
          "map": "forest",
          "mode": "ranked"
        },
        "participants": [
          {
            "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
            "username": "super_player",
            "score": 150,
            "rank_position": 1
          }
        ]
      }
    ],
    "next_cursor": "page_token_string"
  }
  ```

#### 2. Get Match Details
Retrieves detail info on a specific past match.
* **Endpoint**: `GET /v2/match/{match_id}`
* **Response Body (200 OK)**: Same structure as a single match item in List Player Match History.

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/timestamp.proto";

service MatchHistoryService {
  rpc ListMatchHistory(ListMatchHistoryRequest) returns (MatchHistoryList);
  rpc GetMatchDetails(GetMatchDetailsRequest) returns (MatchHistoryRecord);
}

message ListMatchHistoryRequest {
  string user_id = 1;
  int32 limit = 2;
  string cursor = 3;
}

message ParticipantStats {
  string user_id = 1;
  string username = 2;
  int64 score = 3;
  int32 rank_position = 4;
}

message MatchHistoryRecord {
  string match_id = 1;
  string match_type = 2;
  string winner_id = 3;
  int32 duration_sec = 4;
  google.protobuf.Timestamp start_time = 5;
  google.protobuf.Timestamp end_time = 6;
  string metadata = 7; // JSON string format
  repeated ParticipantStats participants = 8;
}

message MatchHistoryList {
  repeated MatchHistoryRecord matches = 1;
  string next_cursor = 2;
}

message GetMatchDetailsRequest {
  string match_id = 1;
}
```

---

### Server Runtime API

```lua
-- Save match state snapshot during active play (e.g. checkpoint)
nk.match_state_save(match_id, state_data, tick, metadata)

-- Load a saved state to resume or analyze
local state_data, tick, metadata = nk.match_state_load(match_id, tick)

-- Write a completed match history record to the ledger database
nk.match_history_write(match_id, match_type, winner_id, duration_sec, start_time, end_time, metadata, participants)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `match.history_retention_days` | integer | `90` | Expiry duration for persistent match history metadata (90 days). |
| `match.snapshot_retention_days` | integer | `7` | Expiry duration for raw state snapshots/checkpoints (7 days). |

---

## Linked Documents
- [BRD-15](../BRD/15_match_state_persistence.md) (Business Requirements Document)
- [TDD-15](../TDD/15_match_state_persistence.md) (Technical Design Document)
