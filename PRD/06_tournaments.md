# PRD-06: Tournaments

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Tournaments  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a tournament system that supports scheduled competitive events with entry limits, durations, multiple seasons, and rewards. Tournaments build on the leaderboard system to provide time-bounded competitive experiences.

---

Refer to [BRD-06](../BRD/06_tournaments.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Tournaments support REST and gRPC. Event state transitions, entry criteria validation, and reward distribution are managed by the server runtime.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Create Tournament (Admin API / Internal)
* **Endpoint**: `POST /v2/tournament`
* **Request Body**:
  ```json
  {
    "id": "weekend_arena",
    "title": "Weekend Arena",
    "description": "Weekly PvP Event",
    "category": 1,
    "sort_order": "descending",
    "operator": "best",
    "duration": 194360,
    "reset_schedule": "0 18 * * 5",
    "start_time": 1782928800,
    "end_time": 1814457600,
    "max_size": 10000,
    "max_num_score": 3,
    "metadata": {
      "rewards_scheme": "arena_v1"
    },
    "authoritative": true
  }
  ```
* **Response Status**: `200 OK` (no body)

#### 2. Delete Tournament (Admin API / Internal)
* **Endpoint**: `DELETE /v2/tournament/{id}`
* **Response Status**: `200 OK` (no body)

#### 3. List Tournaments
* **Endpoint**: `GET /v2/tournament`
* **Query Parameters**:
  * `category_start` (integer, optional): Start category filtering.
  * `category_end` (integer, optional): End category filtering.
  * `active` (boolean, optional): If `true`, only list currently active events.
  * `limit` (integer, optional, default: `100`): Max results.
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**:
  ```json
  {
    "tournaments": [
      {
        "id": "weekend_arena",
        "title": "Weekend Arena",
        "description": "Weekly PvP Event",
        "category": 1,
        "size": 4210,
        "max_size": 10000,
        "start_time": 1782928800,
        "end_time": 1814457600,
        "duration": 194360,
        "next_reset_time": 1783533600,
        "metadata": "{\"rewards_scheme\":\"arena_v1\"}"
      }
    ],
    "next_cursor": "page_token_string"
  }
  ```

#### 4. Join Tournament
* **Endpoint**: `POST /v2/tournament/{id}/join`
* **Response Status**: `200 OK` (no body)

#### 5. Submit Tournament Score
* **Endpoint**: `POST /v2/tournament/{id}`
* **Request Body**:
  ```json
  {
    "score": 4200,
    "subscore": 12,
    "metadata": {
      "win": true
    }
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "leaderboard_id": "weekend_arena",
    "owner_id": "e932b70f-152e-436d-9614-22b274be59c0",
    "username": "super_player",
    "score": 4200,
    "subscore": 12,
    "metadata": "{\"win\":true}",
    "rank": 142,
    "create_time": "2026-07-01T22:34:00Z",
    "update_time": "2026-07-01T22:36:00Z"
  }
  ```

#### 6. List Tournament Records
* **Endpoint**: `GET /v2/tournament/{id}`
* **Query Parameters**: Same as Leaderboards (`limit`, `cursor`, `owner_ids`).
* **Response Body (200 OK)**: Same structure as `GET /v2/leaderboard/{id}`.

#### 7. Around-Player Lookup
* **Endpoint**: `GET /v2/tournament/{id}/around/{owner_id}`
* **Query Parameters**: Same as Leaderboards (`limit`).
* **Response Body (200 OK)**: Same structure as `/v2/leaderboard/{id}/around/{owner_id}`.

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";
import "ultimate/server/api/leaderboard.proto"; // references LeaderboardRecord

service TournamentService {
  rpc CreateTournament(CreateTournamentRequest) returns (google.protobuf.Empty);
  rpc DeleteTournament(DeleteTournamentRequest) returns (google.protobuf.Empty);
  rpc ListTournaments(ListTournamentsRequest) returns (TournamentList);
  rpc JoinTournament(JoinTournamentRequest) returns (google.protobuf.Empty);
  rpc WriteTournamentRecord(WriteTournamentRecordRequest) returns (LeaderboardRecord);
  rpc ListTournamentRecords(ListTournamentRecordsRequest) returns (LeaderboardRecordList);
  rpc ListTournamentRecordsAroundOwner(ListTournamentRecordsAroundOwnerRequest) returns (LeaderboardRecordList);
}

message CreateTournamentRequest {
  string id = 1;
  string title = 2;
  string description = 3;
  int32 category = 4;
  string sort_order = 5;
  string operator = 6;
  int32 duration = 7;
  string reset_schedule = 8;
  google.protobuf.Timestamp start_time = 9;
  google.protobuf.Timestamp end_time = 10;
  int32 max_size = 11;
  int32 max_num_score = 12;
  string metadata = 13;
  bool authoritative = 14;
}

message DeleteTournamentRequest {
  string id = 1;
}

message ListTournamentsRequest {
  int32 category_start = 1;
  int32 category_end = 2;
  bool active = 3;
  int32 limit = 4;
  string cursor = 5;
}

message Tournament {
  string id = 1;
  string title = 2;
  string description = 3;
  int32 category = 4;
  int32 size = 5;
  int32 max_size = 6;
  google.protobuf.Timestamp start_time = 7;
  google.protobuf.Timestamp end_time = 8;
  int32 duration = 9;
  google.protobuf.Timestamp next_reset_time = 10;
  string metadata = 11;
}

message TournamentList {
  repeated Tournament tournaments = 1;
  string next_cursor = 2;
}

message JoinTournamentRequest {
  string tournament_id = 1;
}

message WriteTournamentRecordRequest {
  string tournament_id = 1;
  int64 score = 2;
  int64 subscore = 3;
  string metadata = 4;
}

message ListTournamentRecordsRequest {
  string tournament_id = 1;
  int32 limit = 2;
  string cursor = 3;
  repeated string owner_ids = 4;
}

message ListTournamentRecordsAroundOwnerRequest {
  string tournament_id = 1;
  string owner_id = 2;
  int32 limit = 3;
}
```

---

### Server Runtime API

```lua
nk.tournament_create(id, sort_order, operator, reset_schedule, metadata, title, description, category, start_time, end_time, duration, max_size, max_num_score, authoritative)
nk.tournament_join(id, owner_id, username)
nk.tournament_record_write(id, owner_id, username, score, subscore, metadata)
nk.tournament_records_list(id, owner_ids, limit, cursor, expiry_time)
nk.tournament_records_around_owner(id, owner_id, limit, expiry_time)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `tournament.max_tournaments_active` | integer | `1000` | Hard cap on active tournaments running concurrently. |
| `tournament.state_check_interval_sec`| integer| `1` | Evaluation interval for tournament state updates (cron transitions). |

---

## Linked Documents
- [BRD-06](../BRD/06_tournaments.md) (Business Requirements Document)
- [TDD-06](../TDD/06_tournaments.md) (Technical Design Document)
