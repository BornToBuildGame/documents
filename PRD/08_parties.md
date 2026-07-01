# PRD-08: Parties

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Parties  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a party system enabling players to form temporary groups for co-op play, party matchmaking, party chat, and coordinated game activities. Parties are session-based groups that exist while members are online.

---

Refer to [BRD-08](../BRD/08_parties.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

The party system is entirely real-time and session-based, operating exclusively over the persistent WebSocket connection. All operations are transmitted as socket messages and broadcast to party members.

### Common Socket Envelope
All client-to-server and server-to-client party actions utilize the WebSocket message envelope (see [PRD-03](./03_realtime_multiplayer.md)).

---

## 3. API Surface Detail

### WebSocket Messages

#### 1. Create Party (Client -> Server)
* **Type Name**: `party_create`
* **Payload**:
  ```json
  {
    "open": false,
    "max_size": 4
  }
  ```

#### 2. Join Party (Client -> Server)
* **Type Name**: `party_join`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1"
  }
  ```

#### 3. Leave Party (Client -> Server)
* **Type Name**: `party_leave`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1"
  }
  ```

#### 4. Send Party Invite (Client -> Server)
* **Type Name**: `party_invite`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1",
    "user_id": "bc33910c-992e-46aa-b2b9-111122223333"
  }
  ```

#### 5. Kick Party Member (Client -> Server)
* **Type Name**: `party_kick`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1",
    "presence": {
      "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
      "session_id": "session-uuid-2"
    }
  }
  ```

#### 6. Promote Party Member (Client -> Server)
* **Type Name**: `party_promote`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1",
    "presence": {
      "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
      "session_id": "session-uuid-2"
    }
  }
  ```

#### 7. Send Party Data (Client -> Server)
Used for party chat or custom state coordination (e.g. ready check).
* **Type Name**: `party_data_send`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1",
    "op_code": 1,
    "data": "eyJwYXJ0eV9jaGF0X21zZyI6ICJIZWxsbyEifQ=="
  }
  ```

#### 8. Party Broadcast Update (Server -> Client)
Sent to all members on changes to party status, settings, or membership.
* **Type Name**: `party`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1",
    "open": false,
    "max_size": 4,
    "self": {
      "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
      "username": "super_player",
      "session_id": "session-uuid-1"
    },
    "leader": {
      "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
      "username": "super_player",
      "session_id": "session-uuid-1"
    },
    "presences": [
      {
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "username": "super_player",
        "session_id": "session-uuid-1"
      },
      {
        "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "username": "shadow_ninja",
        "session_id": "session-uuid-2"
      }
    ]
  }
  ```

#### 9. Party Presence Event (Server -> Client)
* **Type Name**: `party_presence_event`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1",
    "joins": [
      {
        "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "username": "shadow_ninja",
        "session_id": "session-uuid-2"
      }
    ],
    "leaves": []
  }
  ```

#### 10. Incoming Party Invite (Server -> Client)
* **Type Name**: `party_invite_incoming`
* **Payload**:
  ```json
  {
    "party_id": "party-session-uuid-1",
    "sender": {
      "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
      "username": "super_player"
    },
    "expiry_time": 1782928860
  }
  ```

---

### Server Runtime API

```lua
nk.party_create(open, max_size)
nk.party_join(party_id, session_id)
nk.party_leave(party_id, session_id)
nk.party_kick(party_id, session_id)
nk.party_promote(party_id, session_id)
nk.party_data_send(party_id, op_code, data)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `party.default_max_size` | integer | `4` | Default limit on members per party. |
| `party.absolute_max_size` | integer | `16` | Absolute limit on members per party. |
| `party.invite_expiry_sec` | integer | `60` | Duration before pending party invites expire. |
| `party.disconnect_grace_sec` | integer | `30` | Duration a disconnected member has to reconnect before being kicked. |

---

## Linked Documents
- [BRD-08](../BRD/08_parties.md) (Business Requirements Document)
- [TDD-08](../TDD/08_parties.md) (Technical Design Document)
