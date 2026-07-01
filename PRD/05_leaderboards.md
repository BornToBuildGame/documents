# PRD-05: Leaderboards

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Leaderboards  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a built-in leaderboard system supporting multiple timeframes, ranking strategies, pagination, and around-player lookups. Leaderboards drive competitive engagement and provide visible progression for players.

---

Refer to [BRD-05](../BRD/05_leaderboards.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Leaderboards support REST and gRPC. Score submissions can be performed directly by clients (unless configured as `authoritative`) or via the server runtime from authoritative matches.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Create Leaderboard (Admin API / Internal)
* **Endpoint**: `POST /v2/leaderboard`
* **Request Body**:
  ```json
  {
    "id": "weekly_highscore",
    "sort_order": "descending",
    "operator": "best",
    "reset_schedule": "0 0 * * 1",
    "metadata": {
      "display_title": "Weekly High Scores"
    },
    "authoritative": true,
    "bucket_size": 100
  }
  ```
* **Response Status**: `200 OK` (no body)

#### 2. Delete Leaderboard (Admin API / Internal)
* **Endpoint**: `DELETE /v2/leaderboard/{id}`
* **Response Status**: `200 OK` (no body)

#### 3. Submit Score
* **Endpoint**: `POST /v2/leaderboard/{id}`
* **Request Body**:
  ```json
  {
    "score": 15600,
    "subscore": 340,
    "metadata": {
      "character": "mage",
      "level_played": "dungeon_1"
    }
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "leaderboard_id": "weekly_highscore",
    "owner_id": "e932b70f-152e-436d-9614-22b274be59c0",
    "username": "super_player",
    "score": 15600,
    "subscore": 340,
    "metadata": "{\"character\":\"mage\",\"level_played\":\"dungeon_1\"}",
    "rank": 4,
    "create_time": "2026-07-01T22:34:00Z",
    "update_time": "2026-07-01T22:35:00Z"
  }
  ```

#### 4. List Leaderboard Records
* **Endpoint**: `GET /v2/leaderboard/{id}`
* **Query Parameters**:
  * `limit` (integer, optional, default: `10`): Max records to retrieve (max 100).
  * `cursor` (string, optional): Pagination token.
  * `owner_ids` (string, optional): Comma-separated list of user IDs to retrieve specifically.
* **Response Body (200 OK)**:
  ```json
  {
    "records": [
      {
        "leaderboard_id": "weekly_highscore",
        "owner_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "username": "super_player",
        "score": 15600,
        "subscore": 340,
        "metadata": "{\"character\":\"mage\",\"level_played\":\"dungeon_1\"}",
        "rank": 4,
        "create_time": "2026-07-01T22:34:00Z",
        "update_time": "2026-07-01T22:35:00Z"
      }
    ],
    "next_cursor": "pagination_token_string",
    "prev_cursor": "pagination_token_string"
  }
  ```

#### 5. Get Owner Record
* **Endpoint**: `GET /v2/leaderboard/{id}/owner/{owner_id}`
* **Response Body (200 OK)**: Same as a single record.

#### 6. Around-Player Lookup
* **Endpoint**: `GET /v2/leaderboard/{id}/around/{owner_id}`
* **Query Parameters**:
  * `limit` (integer, optional, default: `10`): Number of surrounding records (gets split above/below).
* **Response Body (200 OK)**: Same structure as List Leaderboard Records, returning the player and their neighbors.

#### 7. Delete Record
* **Endpoint**: `DELETE /v2/leaderboard/{id}/owner/{owner_id}`
* **Response Status**: `200 OK` (no body)

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";

service LeaderboardService {
  rpc CreateLeaderboard(CreateLeaderboardRequest) returns (google.protobuf.Empty);
  rpc DeleteLeaderboard(DeleteLeaderboardRequest) returns (google.protobuf.Empty);
  rpc WriteLeaderboardRecord(WriteLeaderboardRecordRequest) returns (LeaderboardRecord);
  rpc ListLeaderboardRecords(ListLeaderboardRecordsRequest) returns (LeaderboardRecordList);
  rpc ListLeaderboardRecordsAroundOwner(ListLeaderboardRecordsAroundOwnerRequest) returns (LeaderboardRecordList);
  rpc DeleteLeaderboardRecord(DeleteLeaderboardRecordRequest) returns (google.protobuf.Empty);
}

message CreateLeaderboardRequest {
  string id = 1;
  string sort_order = 2; // "ascending" or "descending"
  string operator = 3;   // "best", "set", or "incr"
  string reset_schedule = 4; // Cron format
  string metadata = 5; // JSON string format
  bool authoritative = 6;
  int32 bucket_size = 7;
}

message DeleteLeaderboardRequest {
  string id = 1;
}

message WriteLeaderboardRecordRequest {
  string leaderboard_id = 1;
  int64 score = 2;
  int64 subscore = 3;
  string metadata = 4; // JSON string format
}

message LeaderboardRecord {
  string leaderboard_id = 1;
  string owner_id = 2;
  string username = 3;
  int64 score = 4;
  int64 subscore = 5;
  string metadata = 6; // JSON string format
  int64 rank = 7;
  google.protobuf.Timestamp create_time = 8;
  google.protobuf.Timestamp update_time = 9;
}

message ListLeaderboardRecordsRequest {
  string leaderboard_id = 1;
  int32 limit = 2;
  string cursor = 3;
  repeated string owner_ids = 4;
}

message ListLeaderboardRecordsAroundOwnerRequest {
  string leaderboard_id = 1;
  string owner_id = 2;
  int32 limit = 3;
}

message LeaderboardRecordList {
  repeated LeaderboardRecord records = 1;
  string next_cursor = 2;
  string prev_cursor = 3;
}

message DeleteLeaderboardRecordRequest {
  string leaderboard_id = 1;
  string owner_id = 2;
}
```

---

### Server Runtime API

```lua
nk.leaderboard_create(id, sort_order, operator, reset_schedule, metadata, authoritative)
nk.leaderboard_record_write(id, owner_id, username, score, subscore, metadata)
nk.leaderboard_records_list(id, owner_ids, limit, cursor, expiry_time)
nk.leaderboard_records_around_owner(id, owner_id, limit, expiry_time)
nk.leaderboard_record_delete(id, owner_id)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `leaderboard.max_records_limit` | integer | `100` | Maximum record count per API query page. |
| `leaderboard.cache_top_n_records` | integer | `1000` | Cache top records count in-memory for zero-latency retrieval. |
| `leaderboard.default_bucket_size` | integer | `0` | Size of brackets for bucketed leaderboards (`0` means disabled). |

---

## Linked Documents
- [BRD-05](../BRD/05_leaderboards.md) (Business Requirements Document)
- [TDD-05](../TDD/05_leaderboards.md) (Technical Design Document)
