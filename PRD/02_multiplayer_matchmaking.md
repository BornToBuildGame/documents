# PRD-02: Multiplayer Matchmaking

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Multiplayer Matchmaking  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a flexible matchmaking system that pairs players into compatible game sessions based on skill, region, game mode, and custom properties. The system supports solo players, parties, and custom matchmaking logic.

---

Refer to [BRD-02](../BRD/02_multiplayer_matchmaking.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Matchmaking supports REST, WebSocket, and gRPC. Due to the asynchronous nature of matchmaking, WebSocket connections are the recommended transport to receive immediate push notifications when a match is found. Polling via REST/gRPC is supported as a fallback.

### WebSocket Frame Format
All client WebSocket messages use the common envelope:
```json
{
  "cid": "correlation_id_123",
  "matchmaker_add": {
    "queue_name": "ranked_5v5",
    "query": "+properties.region:us-east +properties.mode:ranked",
    "min_count": 10,
    "max_count": 10,
    "string_properties": {
      "region": "us-east",
      "mode": "ranked"
    },
    "numeric_properties": {
      "skill": 1500.0
    }
  }
}
```

Server-to-client notifications are delivered over the same WebSocket connection.

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Submit Matchmaking Ticket
* **Endpoint**: `POST /v2/matchmaker/ticket`
* **Request Headers**: `Authorization: Bearer <session_token>`
* **Request Body**:
  ```json
  {
    "queue_name": "ranked_5v5",
    "min_count": 10,
    "max_count": 10,
    "string_properties": {
      "region": "us-east",
      "mode": "ranked"
    },
    "numeric_properties": {
      "skill": 1500.0
    },
    "count_multiple": 2,
    "reverse_precision": true
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "ticket_id": "8fa8d1df-2e11-47cc-9831-50e501254124",
    "queue_name": "ranked_5v5",
    "created_at": "2026-07-01T22:34:00Z"
  }
  ```

#### 2. Cancel Matchmaking Ticket
* **Endpoint**: `DELETE /v2/matchmaker/ticket/{ticket_id}`
* **Request Headers**: `Authorization: Bearer <session_token>`
* **Response Status**: `200 OK` (no body)

#### 3. Get Matchmaking Ticket Status
* **Endpoint**: `GET /v2/matchmaker/ticket/{ticket_id}`
* **Request Headers**: `Authorization: Bearer <session_token>`
* **Response Body (200 OK)**:
  ```json
  {
    "ticket_id": "8fa8d1df-2e11-47cc-9831-50e501254124",
    "queue_name": "ranked_5v5",
    "status": "queued",
    "created_at": "2026-07-01T22:34:00Z",
    "match_id": ""
  }
  ```

#### 4. Get Queue Statistics
* **Endpoint**: `GET /v2/matchmaker/stats/{queue_name}`
* **Request Headers**: `Authorization: Bearer <session_token>`
* **Response Body (200 OK)**:
  ```json
  {
    "queue_name": "ranked_5v5",
    "ticket_count": 412,
    "average_wait_sec": 24,
    "active_matches": 15
  }
  ```

---

### WebSocket Messages

#### 1. Add to Matchmaker (Client -> Server)
* **Type Name**: `matchmaker_add`
* **Payload**:
  ```json
  {
    "queue_name": "ranked_5v5",
    "min_count": 10,
    "max_count": 10,
    "string_properties": {
      "region": "us-east"
    },
    "numeric_properties": {
      "skill": 1450.0
    }
  }
  ```

#### 2. Remove from Matchmaker (Client -> Server)
* **Type Name**: `matchmaker_remove`
* **Payload**:
  ```json
  {
    "ticket_id": "8fa8d1df-2e11-47cc-9831-50e501254124"
  }
  ```

#### 3. Ticket Status Update (Server -> Client)
* **Type Name**: `matchmaker_ticket`
* **Payload**:
  ```json
  {
    "ticket_id": "8fa8d1df-2e11-47cc-9831-50e501254124",
    "queue_name": "ranked_5v5",
    "status": "queued",
    "created_at": "2026-07-01T22:34:00Z"
  }
  ```

#### 4. Match Found Notification (Server -> Client)
* **Type Name**: `matchmaker_matched`
* **Payload**:
  ```json
  {
    "ticket_id": "8fa8d1df-2e11-47cc-9831-50e501254124",
    "match_id": "authoritative-match-uuid-111",
    "match_token": "secure-match-join-token-jwt...",
    "presences": [
      {
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "username": "super_player",
        "session_id": "session-uuid-222"
      },
      {
        "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "username": "shadow_ninja",
        "session_id": "session-uuid-333"
      }
    ],
    "self": {
      "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
      "username": "super_player",
      "session_id": "session-uuid-222"
    }
  }
  ```

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/empty.proto";

service MatchmakerService {
  rpc AddMatchmaker(AddMatchmakerRequest) returns (MatchmakerTicket);
  rpc RemoveMatchmaker(RemoveMatchmakerRequest) returns (google.protobuf.Empty);
  rpc GetMatchmakerTicket(GetMatchmakerTicketRequest) returns (MatchmakerTicket);
  rpc GetQueueStats(GetQueueStatsRequest) returns (QueueStats);
}

message AddMatchmakerRequest {
  string queue_name = 1;
  int32 min_count = 2;
  int32 max_count = 3;
  map<string, string> string_properties = 4;
  map<string, double> numeric_properties = 5;
  int32 count_multiple = 6;
  bool reverse_precision = 7;
}

message MatchmakerTicket {
  string ticket_id = 1;
  string queue_name = 2;
  string status = 3; // "queued", "matched", "cancelled", "expired"
  string match_id = 4; // empty if not matched yet
  string match_token = 5; // empty if not matched yet
}

message RemoveMatchmakerRequest {
  string ticket_id = 1;
}

message GetMatchmakerTicketRequest {
  string ticket_id = 1;
}

message GetQueueStatsRequest {
  string queue_name = 1;
}

message QueueStats {
  string queue_name = 2;
  int32 ticket_count = 3;
  int32 average_wait_sec = 4;
  int32 active_matches = 5;
}
```

---

## 4. Product Configurations

Matchmaking queues are defined in the central configuration file.

```yaml
matchmaker:
  processing_interval_ms: 1000 # processing loop frequency (1s)
  ticket_expiry_sec: 300       # ticket automatic expiry (5m)
  queues:
    ranked_5v5:
      min_players: 10
      max_players: 10
      skill_match:
        enabled: true
        initial_range: 50.0
        max_range: 500.0
        expansion_interval_sec: 5
        expansion_step: 25.0
      region_match:
        strict: false
        fallback_delay_sec: 15
      reverse_precision:
        enabled: true
        reverse_threshold_ticks: 10
    casual_2v2:
      min_players: 4
      max_players: 4
      count_multiple: 2
      skill_match:
        enabled: false
```

---

## 5. Linked Documents
- [BRD-02](../BRD/02_multiplayer_matchmaking.md) (Business Requirements Document)
- [TDD-02](../TDD/02_multiplayer_matchmaking.md) (Technical Design Document)
