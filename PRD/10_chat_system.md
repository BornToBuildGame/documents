# PRD-10: Chat System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Chat System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a comprehensive real-time chat system supporting direct messages, group channels, guild channels, party channels, match channels, and custom global channels. The system supports persistent history, moderation, and presence tracking.

---

Refer to [BRD-10](../BRD/10_chat_system.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Chat messages are sent and received in real-time over persistent WebSocket connections. Fetching historical messages from persistent channels can be performed over HTTP/REST or gRPC.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### WebSocket Messages

#### 1. Join Chat Channel (Client -> Server)
* **Type Name**: `channel_join`
* **Payload**:
  ```json
  {
    "target": "guild-uuid-1111-2222",
    "type": 3,
    "persistence": true,
    "hidden": false
  }
  ```
  *(Note: `type` corresponds to: `1` = Direct Message, `2` = Group, `3` = Guild, `4` = Party, `5` = Match, `6` = Custom. `target` is the ID of the corresponding entity or the user ID for DMs).*
* **Server Response (delivered to joining client)**:
  ```json
  {
    "cid": "client-correlation-id",
    "channel": {
      "id": "channel-runtime-id-string",
      "presences": [
        {
          "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
          "username": "super_player"
        }
      ],
      "self": {
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "username": "super_player"
      }
    }
  }
  ```

#### 2. Leave Chat Channel (Client -> Server)
* **Type Name**: `channel_leave`
* **Payload**:
  ```json
  {
    "channel_id": "channel-runtime-id-string"
  }
  ```

#### 3. Send Message (Client -> Server)
* **Type Name**: `channel_message_send`
* **Payload**:
  ```json
  {
    "channel_id": "channel-runtime-id-string",
    "content": {
      "text": "Hello, Guild!"
    }
  }
  ```

#### 4. Receive Chat Message (Server -> Client)
* **Type Name**: `channel_message`
* **Payload**:
  ```json
  {
    "channel_id": "channel-runtime-id-string",
    "message_id": "message-uuid-5555",
    "code": 0,
    "sender_id": "e932b70f-152e-436d-9614-22b274be59c0",
    "username": "super_player",
    "content": {
      "text": "Hello, Guild!"
    },
    "create_time": "2026-07-01T22:30:00Z",
    "update_time": "2026-07-01T22:30:00Z",
    "persistent": true
  }
  ```

---

### REST Endpoints

#### 1. Retrieve Channel Message History
* **Endpoint**: `GET /v2/channel/{channel_id}`
* **Query Parameters**:
  * `limit` (integer, optional, default: `20`, max: `100`): Message count.
  * `forward` (boolean, optional, default: `false`): If `true`, lists oldest-to-newest; else newest-to-oldest.
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**:
  ```json
  {
    "messages": [
      {
        "channel_id": "channel-runtime-id-string",
        "message_id": "message-uuid-5555",
        "code": 0,
        "sender_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "username": "super_player",
        "content": "{\"text\":\"Hello, Guild!\"}",
        "create_time": "2026-07-01T22:30:00Z",
        "update_time": "2026-07-01T22:30:00Z",
        "persistent": true
      }
    ],
    "next_cursor": "history_page_token",
    "prev_cursor": "history_page_token"
  }
  ```

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/timestamp.proto";

service ChatService {
  rpc ListChannelMessages(ListChannelMessagesRequest) returns (ChannelMessageList);
}

message ListChannelMessagesRequest {
  string channel_id = 1;
  int32 limit = 2;
  bool forward = 3;
  string cursor = 4;
}

message ChannelMessage {
  string channel_id = 1;
  string message_id = 2;
  int32 code = 3;
  string sender_id = 4;
  string username = 5;
  string content = 6; // JSON string format
  google.protobuf.Timestamp create_time = 7;
  google.protobuf.Timestamp update_time = 8;
  bool persistent = 9;
}

message ChannelMessageList {
  repeated ChannelMessage messages = 1;
  string next_cursor = 2;
  string prev_cursor = 3;
}
```

---

### Server Runtime API

```lua
nk.channel_message_send(channel_id, content, sender_id, sender_username, persist)
nk.channel_message_list(channel_id, limit, forward, cursor)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `chat.max_message_size_bytes` | integer | `4096` | Max size of individual chat message payload (4 KB). |
| `chat.message_retention_days` | integer | `30` | Duration to keep persistent chat records in database. |
| `chat.rate_limit_per_user_sec` | integer | `5` | Maximum messages allowed per user per second. |

---

## Linked Documents
- [BRD-10](../BRD/10_chat_system.md) (Business Requirements Document)
- [TDD-10](../TDD/10_chat_system.md) (Technical Design Document)
