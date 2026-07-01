# PRD-11: Notifications

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Notifications  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a server-driven notification system that delivers both real-time and persistent notifications to players. Notifications cover friend requests, tournament rewards, daily bonuses, game events, and custom developer-defined alerts.

---

Refer to [BRD-11](../BRD/11_notifications.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Notifications are delivered in real-time as WebSocket frames if the client is connected. Persistent notifications are stored and can be queried or modified via REST or gRPC.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. List Notifications
* **Endpoint**: `GET /v2/notification`
* **Query Parameters**:
  * `limit` (integer, optional, default: `20`, max: `100`)
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**:
  ```json
  {
    "notifications": [
      {
        "id": "notification-uuid-9999",
        "subject": "Friend Request",
        "content": "{\"user_id\":\"bc33910c-992e-46aa-b2b9-111122223333\",\"username\":\"shadow_ninja\"}",
        "code": 100,
        "sender_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "create_time": "2026-07-01T22:30:00Z",
        "persistent": true
      }
    ],
    "cacheable_cursor": "page_token_string"
  }
  ```

#### 2. Delete / Acknowledge Notifications
* **Endpoint**: `DELETE /v2/notification`
* **Query Parameters**:
  * `ids` (string, required): Comma-separated list of notification IDs to delete/acknowledge.
* **Response Status**: `200 OK` (no body)

---

### WebSocket Messages

#### 1. Real-time Notification Broadcast (Server -> Client)
Delivered immediately when a notification is generated while the user is online.
* **Type Name**: `notifications`
* **Payload**:
  ```json
  {
    "notifications": [
      {
        "id": "notification-uuid-9999",
        "subject": "Friend Request",
        "content": "{\"user_id\":\"bc33910c-992e-46aa-b2b9-111122223333\",\"username\":\"shadow_ninja\"}",
        "code": 100,
        "sender_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "create_time": "2026-07-01T22:30:00Z",
        "persistent": true
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
import "google/protobuf/timestamp.proto";

service NotificationService {
  rpc ListNotifications(ListNotificationsRequest) returns (NotificationList);
  rpc DeleteNotifications(DeleteNotificationsRequest) returns (google.protobuf.Empty);
}

message ListNotificationsRequest {
  int32 limit = 1;
  string cursor = 2;
}

message Notification {
  string id = 1;
  string subject = 2;
  string content = 3; // JSON string format
  int32 code = 4;
  string sender_id = 5;
  google.protobuf.Timestamp create_time = 6;
  bool persistent = 7;
}

message NotificationList {
  repeated Notification notifications = 1;
  string cacheable_cursor = 2;
}

message DeleteNotificationsRequest {
  repeated string ids = 1;
}
```

---

### Server Runtime API

```lua
nk.notification_send(user_id, subject, content, code, sender_id, persistent)
nk.notifications_send(notifications) -- array of notification specs for batch processing
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `notification.expiry_days` | integer | `30` | Number of days persistent notifications are kept before being auto-deleted. |
| `notification.max_persistent_per_user` | integer | `1000` | Hard cap on unread persistent notifications stored per user. |

---

## Linked Documents
- [BRD-11](../BRD/11_notifications.md) (Business Requirements Document)
- [TDD-11](../TDD/11_notifications.md) (Technical Design Document)
