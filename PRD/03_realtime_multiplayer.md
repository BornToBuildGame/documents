# PRD-03: Realtime Multiplayer

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Realtime Multiplayer  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a realtime multiplayer infrastructure that supports low-latency game state synchronization, presence tracking, and reliable/unreliable message delivery over persistent connections. This system enables a wide range of game genres including FPS, MOBA, RTS, racing, board games, and card games.

---

Refer to [BRD-03](../BRD/03_realtime_multiplayer.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Realtime multiplayer utilizes persistent WebSocket (RFC 6455) connections running over TLS (WSS) by default. The protocol utilizes a standardized JSON envelope structure for control messages, while state data transmission (`match_data_send` / `match_data`) supports binary payloads (or base64 encoded text in JSON envelopes).

### WebSocket Handshake
* **Endpoint**: `GET /v2/stream`
* **Query Parameters**:
  * `token` (string, required): Session JWT token.
  * `status` (boolean, optional, default: `true`): Appear online automatically.
* **Headers**:
  * `Sec-WebSocket-Protocol`: `ultimate-json` or `ultimate-binary`

### Common WebSocket Envelope
All client-to-server and server-to-client WebSocket messages use the common envelope:
```json
{
  "cid": "client-correlation-id-string",
  "payload_field": {
    "data_fields": "values"
  }
}
```

---

## 3. API Surface Detail

### REST Endpoints

#### 1. List Active Matches
* **Endpoint**: `GET /v2/match`
* **Query Parameters**:
  * `limit` (integer, optional, default: `100`, max: `1000`): Maximum matches to return.
  * `authoritative` (boolean, optional): Filter by server-authoritative matches.
  * `label` (string, optional): Filter matches by a dynamic label query (e.g. `+mode:ranked`).
  * `min_size` (integer, optional)
  * `max_size` (integer, optional)
* **Response Body (200 OK)**:
  ```json
  {
    "matches": [
      {
        "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
        "authoritative": true,
        "label": "{\"mode\":\"ranked\",\"map\":\"forest\",\"players\":4}",
        "size": 4,
        "max_size": 10
      }
    ]
  }
  ```

#### 2. Get Match Details
* **Endpoint**: `GET /v2/match/{match_id}`
* **Response Body (200 OK)**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
    "authoritative": true,
    "label": "{\"mode\":\"ranked\",\"map\":\"forest\",\"players\":4}",
    "size": 4,
    "max_size": 10,
    "presences": [
      {
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "username": "super_player",
        "session_id": "session-uuid-1"
      }
    ]
  }
  ```

#### 3. Create Authoritative Match (Server-initiated)
* **Endpoint**: `POST /v2/match`
* **Request Body**:
  ```json
  {
    "module": "battle_royale",
    "params": {
      "mode": "ranked",
      "map": "forest"
    }
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124"
  }
  ```

---

### WebSocket Messages

#### 1. Create Relay Match (Client -> Server)
* **Type Name**: `match_create`
* **Payload**:
  ```json
  {}
  ```

#### 2. Join Match (Client -> Server)
* **Type Name**: `match_join`
* **Payload**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
    "token": "optional-matchmaking-match-token",
    "metadata": {
      "player_color": "red"
    }
  }
  ```

#### 3. Leave Match (Client -> Server)
* **Type Name**: `match_leave`
* **Payload**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124"
  }
  ```

#### 4. Send Match Data (Client -> Server)
* **Type Name**: `match_data_send`
* **Payload**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
    "op_code": 101,
    "data": "eyJwIjogeDEyLCB5OjQ1fQ==",
    "reliable": false
  }
  ```
  *(Note: `data` is base64-encoded binary payload. If using binary WebSocket frames, envelope structure is replaced by direct binary payload serialization).*

#### 5. Receive Match Data (Server -> Client)
* **Type Name**: `match_data`
* **Payload**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
    "presence": {
      "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
      "username": "super_player",
      "session_id": "session-uuid-1"
    },
    "op_code": 101,
    "data": "eyJwIjogeDEyLCB5OjQ1fQ=="
  }
  ```

#### 6. Match Presence Event (Server -> Client)
* **Type Name**: `match_presence_event`
* **Payload**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
    "joins": [
      {
        "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "username": "shadow_ninja",
        "session_id": "session-uuid-2"
      }
    ],
    "leaves": [
      {
        "user_id": "0123b70f-442e-1111-9614-22b274be59aa",
        "username": "logged_out_user",
        "session_id": "session-uuid-9"
      }
    ]
  }
  ```

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/empty.proto";

service RealtimeService {
  rpc CreateMatch(CreateMatchRequest) returns (Match);
  rpc ListMatches(ListMatchesRequest) returns (MatchList);
  rpc GetMatch(GetMatchRequest) returns (Match);
}

message CreateMatchRequest {
  string module = 1;
  map<string, string> params = 2;
}

message ListMatchesRequest {
  int32 limit = 1;
  bool authoritative = 2;
  string label = 3;
  int32 min_size = 4;
  int32 max_size = 5;
}

message GetMatchRequest {
  string match_id = 1;
}

message MatchPresence {
  string user_id = 1;
  string username = 2;
  string session_id = 3;
}

message Match {
  string match_id = 1;
  bool authoritative = 2;
  string label = 3;
  int32 size = 4;
  int32 max_size = 5;
  repeated MatchPresence presences = 6;
}

message MatchList {
  repeated Match matches = 1;
}
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `socket.max_message_size_bytes` | integer | `4096` | Maximum size of websocket frames (4 KB). |
| `socket.ping_interval_sec` | integer | `15` | Delay between server sending ping frames (15s). |
| `socket.ping_timeout_sec` | integer | `30` | Time waiting for client pong reply before disconnect. |
| `socket.reconnect_grace_sec` | integer | `30` | Wait duration for a disconnected player to rejoin active match. |
| `socket.write_buffer_size` | integer | `2048` | Message buffer size per websocket connection channel. |

---

## 5. Linked Documents
- [BRD-03](../BRD/03_realtime_multiplayer.md) (Business Requirements Document)
- [TDD-03](../TDD/03_realtime_multiplayer.md) (Technical Design Document)
