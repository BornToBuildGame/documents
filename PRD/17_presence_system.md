# PRD-17: Presence System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Presence System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a real-time presence system that tracks the online status, current activity, match participation, party status, and other live state information of connected users. Presence is foundational to social features, matchmaking, and real-time coordination.

---

Refer to [BRD-17](../BRD/17_presence_system.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Presence tracking is exclusively real-time and runs over the persistent WebSocket connection. Status changes, joins, and leaves are fanned out to active subscribers.

### Common Socket Envelope
All client-to-server and server-to-client presence updates utilize the WebSocket message envelope (see [PRD-03](./03_realtime_multiplayer.md)).

---

## 3. API Surface Detail

### WebSocket Messages

#### 1. Subscribe to User Status (`status_follow`)
* **Type Name**: `status_follow`
* **Payload**:
  ```json
  {
    "user_ids": [
      "bc33910c-992e-46aa-b2b9-111122223333"
    ]
  }
  ```

#### 2. Unsubscribe from User Status (`status_unfollow`)
* **Type Name**: `status_unfollow`
* **Payload**:
  ```json
  {
    "user_ids": [
      "bc33910c-992e-46aa-b2b9-111122223333"
    ]
  }
  ```

#### 3. Update Own Status (`status_update`)
* **Type Name**: `status_update`
* **Payload**:
  ```json
  {
    "status": "in_menu",
    "metadata": {
      "display_text": "Custom status message"
    }
  }
  ```

#### 4. Status Presence Event (Server -> Client)
* **Type Name**: `status_presence_event`
* **Payload**:
  ```json
  {
    "joins": [
      {
        "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "username": "shadow_ninja",
        "status": "in_menu",
        "metadata": {
          "display_text": "Custom status message"
        }
      }
    ],
    "leaves": [
      {
        "user_id": "0123b70f-442e-1111-9614-22b274be59aa",
        "username": "logged_out_user"
      }
    ]
  }
  ```

---

### Server Runtime API

```lua
-- Follow/unfollow target user status
nk.status_follow(user_id, target_ids)
nk.status_unfollow(user_id, target_ids)

-- Update custom status payload for a user
nk.status_update(user_id, status_string)

-- Low-level Stream Management (custom real-time message streams)
-- A stream is a struct: { mode = integer, subject = string, descriptor = string, label = string }
nk.stream_user_list(stream)
nk.stream_user_join(stream, user_id, session_id, hidden, persistence)
nk.stream_user_leave(stream, user_id, session_id)
nk.stream_count(stream)
nk.stream_send(stream, data_string, presences, sender)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `presence.grace_period_sec` | integer | `15` | Delay in seconds before a disconnected user is marked offline. |
| `presence.max_subscriptions_per_user` | integer | `1000` | Hard cap on the number of concurrent status follows allowed per client session. |

---

## Linked Documents
- [BRD-17](../BRD/17_presence_system.md) (Business Requirements Document)
- [TDD-17](../TDD/17_presence_system.md) (Technical Design Document)
