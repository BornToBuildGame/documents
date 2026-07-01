# PRD-04: Authoritative Game Server

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Authoritative Game Server  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for an authoritative game server where game logic executes server-side, preventing cheating and ensuring fair gameplay. The server validates all player actions, owns game state, and distributes results to clients.

---

Refer to [BRD-04](../BRD/04_authoritative_game_server.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Authoritative matches run server-side. Clients interact with the match via persistent WebSocket connections, sending `match_data_send` frames and receiving `match_data` frames. External services or administrators send control requests to the match via REST and gRPC.

---

## 3. API Surface Detail

### Match Handler Interface (Server Runtime)
All custom match handlers registered in the server runtime (Lua, TypeScript, Go) must implement the following lifecycle interface:

#### 1. Match Initialization (`match_init`)
* **Signature**: `match_init(context, params) -> (state, tick_rate, label)`
* **Arguments**:
  * `context`: Environment execution context metadata.
  * `params`: Key-value configuration parameters passed on match creation.
* **Returns**:
  * `state` (object/table): Initial state representation (stored in-memory by server).
  * `tick_rate` (integer): Number of tick updates per second (1 to 60).
  * `label` (string): Dynamic JSON string containing queryable match details.

#### 2. Match Join Attempt (`match_join_attempt`)
* **Signature**: `match_join_attempt(context, dispatcher, tick, state, presence, metadata) -> (state, allow, reject_reason)`
* **Arguments**:
  * `presence`: Object containing `user_id`, `username`, `session_id`.
  * `metadata`: Client-provided metadata (e.g. skin choice, team preference).
* **Returns**:
  * `state`: Updated state.
  * `allow` (boolean): If `true`, player joins the match.
  * `reject_reason` (string): Message returned to client if rejected.

#### 3. Match Join (`match_join`)
* **Signature**: `match_join(context, dispatcher, tick, state, presences) -> state`
* **Arguments**:
  * `presences` (array): List of joined players.

#### 4. Match Leave (`match_leave`)
* **Signature**: `match_leave(context, dispatcher, tick, state, presences) -> state`
* **Arguments**:
  * `presences` (array): List of players who left or disconnected.

#### 5. Match Loop (`match_loop`)
* **Signature**: `match_loop(context, dispatcher, tick, state, messages) -> state (or nil to terminate)`
* **Arguments**:
  * `tick` (integer): Current monotonic tick number of the match.
  * `messages` (array): List of messages received from players since the last tick. Each message contains `sender`, `op_code`, and `data` (binary).
* **Returns**:
  * `state`: Updated state, or `nil` to terminate the match immediately.

#### 6. Match Signal (`match_signal`)
* **Signature**: `match_signal(context, dispatcher, tick, state, data) -> (state, response_data)`
* **Arguments**:
  * `data` (string): Arbitrary data sent from an external system.
* **Returns**:
  * `state`: Updated state.
  * `response_data` (string): Return payload sent back to the signal source.

#### 7. Match Terminate (`match_terminate`)
* **Signature**: `match_terminate(context, dispatcher, tick, state, grace_seconds) -> nil`

---

### Dispatcher Interface (Available within Match Handlers)
Handlers execute side-effects via the `dispatcher` object:

* `dispatcher.broadcast_message(op_code, data, presences, sender)`: Sends game state data to specified presences (or all if `presences` is nil).
* `dispatcher.match_kick(presences)`: Evicts specified players from the match.
* `dispatcher.match_label_update(label)`: Updates the match's listing label dynamically.

---

### External Control APIs

#### 1. Send Signal to Match (REST)
* **Endpoint**: `POST /v2/rpc/match_signal`
* **Request Headers**: `Authorization: Bearer <session_token>`
* **Request Body**:
  ```json
  {
    "match_id": "c1a6be1a-019e-4cda-920f-01a501254124",
    "payload": "{\"command\":\"force_end_game\",\"reason\":\"maintenance\"}"
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "response": "{\"status\":\"ok\",\"players_notified\":12}"
  }
  ```

#### 2. Get Authoritative Match List (REST)
Same as `/v2/match` with `authoritative=true` parameter (see [PRD-03](./03_realtime_multiplayer.md)).

#### 3. gRPC Match Signal Definition
```protobuf
syntax = "proto3";

package ultimate.server.api;

service AuthoritativeMatchService {
  rpc MatchSignal(MatchSignalRequest) returns (MatchSignalResponse);
}

message MatchSignalRequest {
  string match_id = 1;
  string payload = 2;
}

message MatchSignalResponse {
  string response = 1;
}
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `match.default_tick_rate` | integer | `10` | Default game loop ticks per second. |
| `match.max_tick_rate` | integer | `60` | Maximum game loop tick rate permitted. |
| `match.state_size_limit_bytes` | integer | `1048576` | Maximum serialized in-memory match state size (1 MB). |
| `match.handler_execution_timeout_ms`| integer | `50` | Maximum processing duration for a single tick before thread cancellation. |
| `match.empty_terminate_delay_sec` | integer | `0` | Delay before destroying match after all players leave. |

---

## 5. Linked Documents
- [BRD-04](../BRD/04_authoritative_game_server.md) (Business Requirements Document)
- [TDD-04](../TDD/04_authoritative_game_server.md) (Technical Design Document)
