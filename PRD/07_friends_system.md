# PRD-07: Friends System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Friends System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a built-in social graph enabling players to add friends, manage friend requests, block users, and track online status. The friends system is foundational to social features like party formation, direct messaging, and friend leaderboards.

---

Refer to [BRD-07](../BRD/07_friends_system.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

The friends graph modification and retrieval operations run over HTTP/REST and gRPC. Real-time online status and game activity tracking are transmitted over the persistent WebSocket connection.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Send Friend Request / Add Friend
* **Endpoint**: `POST /v2/friend`
* **Request Body**:
  ```json
  {
    "ids": ["bc33910c-992e-46aa-b2b9-111122223333"],
    "usernames": ["shadow_ninja"]
  }
  ```
  *(Note: You can request by `ids`, `usernames`, or both in a single call).*
* **Response Status**: `200 OK` (no body)

#### 2. List Friends (and pending requests/blocked users)
* **Endpoint**: `GET /v2/friend`
* **Query Parameters**:
  * `limit` (integer, optional, default: `100`, max: `1000`): Max entries.
  * `state` (integer, optional): Filter by relationship state:
    * `0`: Mutual friends
    * `1`: Outgoing pending friend requests
    * `2`: Incoming pending friend requests
    * `3`: Blocked users
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**:
  ```json
  {
    "friends": [
      {
        "user": {
          "id": "bc33910c-992e-46aa-b2b9-111122223333",
          "username": "shadow_ninja",
          "display_name": "Shadow Ninja",
          "avatar_url": "https://example.com/ninja.png",
          "lang_tag": "en-US",
          "location": "JP",
          "timezone": "Asia/Tokyo",
          "metadata": "{}"
        },
        "state": 0,
        "update_time": "2026-07-01T22:30:00Z"
      }
    ],
    "next_cursor": "page_token_string"
  }
  ```

#### 3. Remove Friend / Cancel Request
* **Endpoint**: `DELETE /v2/friend/{user_id}`
* **Response Status**: `200 OK` (no body)

#### 4. Block User
* **Endpoint**: `POST /v2/friend/block/{user_id}`
* **Response Status**: `200 OK` (no body)

#### 5. Unblock User
* **Endpoint**: `DELETE /v2/friend/block/{user_id}`
* **Response Status**: `200 OK` (no body)

---

### WebSocket Messages (Presence Tracking)

#### 1. Follow User Status (Client -> Server)
Starts receiving presence events for specified users.
* **Type Name**: `status_follow`
* **Payload**:
  ```json
  {
    "user_ids": ["bc33910c-992e-46aa-b2b9-111122223333"]
  }
  ```

#### 2. Unfollow User Status (Client -> Server)
Stops receiving presence events for specified users.
* **Type Name**: `status_unfollow`
* **Payload**:
  ```json
  {
    "user_ids": ["bc33910c-992e-46aa-b2b9-111122223333"]
  }
  ```

#### 3. Update Own Status (Client -> Server)
* **Type Name**: `status_update`
* **Payload**:
  ```json
  {
    "status": "In Game",
    "metadata": {
      "match_id": "c1a6be1a-019e-4cda-920f-01a501254124"
    }
  }
  ```

#### 4. Status Presence Event (Server -> Client)
* **Type Name**: `status_presence_event`
* **Payload**:
  ```json
  {
    "leaves": [],
    "joins": [
      {
        "user_id": "bc33910c-992e-46aa-b2b9-111122223333",
        "username": "shadow_ninja",
        "status": "In Game",
        "metadata": {
          "match_id": "c1a6be1a-019e-4cda-920f-01a501254124"
        }
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
import "ultimate/server/api/user.proto"; // references User message

service FriendsService {
  rpc AddFriends(AddFriendsRequest) returns (google.protobuf.Empty);
  rpc ListFriends(ListFriendsRequest) returns (FriendList);
  rpc DeleteFriends(DeleteFriendsRequest) returns (google.protobuf.Empty);
  rpc BlockFriends(BlockFriendsRequest) returns (google.protobuf.Empty);
  rpc ImportFacebookFriends(ImportFacebookFriendsRequest) returns (google.protobuf.Empty);
  rpc ImportSteamFriends(ImportSteamFriendsRequest) returns (google.protobuf.Empty);
}

message AddFriendsRequest {
  repeated string ids = 1;
  repeated string usernames = 2;
}

message ListFriendsRequest {
  int32 limit = 1;
  int32 state = 2; // 0=friends, 1=sent request, 2=received request, 3=blocked
  string cursor = 3;
}

message Friend {
  User user = 1;
  int32 state = 2; // 0=friends, 1=sent request, 2=received request, 3=blocked
  google.protobuf.Timestamp update_time = 3;
}

message FriendList {
  repeated Friend friends = 1;
  string next_cursor = 2;
}

message DeleteFriendsRequest {
  repeated string ids = 1;
}

message BlockFriendsRequest {
  repeated string ids = 1;
}

message ImportFacebookFriendsRequest {
  string token = 1;
  bool reset = 2;
}

message ImportSteamFriendsRequest {
  string steam_token = 1;
  bool reset = 2;
}
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `friend.max_count` | integer | `1000` | Maximum number of friends a user is allowed to have. |
| `friend.max_pending` | integer | `100` | Maximum number of incoming/outgoing pending friend requests allowed. |

---

## Linked Documents
- [BRD-07](../BRD/07_friends_system.md) (Business Requirements Document)
- [TDD-07](../TDD/07_friends_system.md) (Technical Design Document)
