# PRD-09: Guilds & Clans

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Guilds & Clans  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a persistent guild/clan system enabling players to form long-lived organizations with membership management, roles, permissions, chat, and metadata. Guilds provide a social structure beyond friends and parties, supporting cooperative gameplay and community building.

---

Refer to [BRD-09](../BRD/09_guilds_clans.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Guild operations run over HTTP/REST and gRPC. Dynamic updates (e.g. member joins, promotions) trigger real-time notification events delivered over the user's active WebSocket connection.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Create Guild (Group)
* **Endpoint**: `POST /v2/group`
* **Request Body**:
  ```json
  {
    "name": "knights_of_honor",
    "description": "A competitive PvP guild.",
    "avatar_url": "https://example.com/emblem.jpg",
    "lang_tag": "en-US",
    "open": false,
    "max_count": 100,
    "metadata": {
      "level_requirement": 20
    }
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "id": "guild-uuid-1111-2222",
    "creator_id": "e932b70f-152e-436d-9614-22b274be59c0",
    "name": "knights_of_honor",
    "description": "A competitive PvP guild.",
    "avatar_url": "https://example.com/emblem.jpg",
    "lang_tag": "en-US",
    "open": false,
    "edge_count": 1,
    "max_count": 100,
    "metadata": "{\"level_requirement\":20}",
    "create_time": "2026-07-01T22:30:00Z",
    "update_time": "2026-07-01T22:30:00Z"
  }
  ```

#### 2. Update Guild Settings
* **Endpoint**: `PUT /v2/group/{id}`
* **Request Body**: Same body format as Create Guild (all fields optional).
* **Response Status**: `200 OK` (no body)

#### 3. Delete Guild
* **Endpoint**: `DELETE /v2/group/{id}`
* **Response Status**: `200 OK` (no body)

#### 4. List / Search Guilds
* **Endpoint**: `GET /v2/group`
* **Query Parameters**:
  * `name` (string, optional): Query filter for guild name (SQL `LIKE` match).
  * `lang_tag` (string, optional): Language filter.
  * `open` (boolean, optional)
  * `limit` (integer, optional, default: `100`): Max results.
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**:
  ```json
  {
    "groups": [
      {
        "id": "guild-uuid-1111-2222",
        "name": "knights_of_honor",
        "description": "A competitive PvP guild.",
        "avatar_url": "https://example.com/emblem.jpg",
        "lang_tag": "en-US",
        "open": false,
        "edge_count": 14,
        "max_count": 100,
        "metadata": "{\"level_requirement\":20}",
        "create_time": "2026-07-01T22:30:00Z",
        "update_time": "2026-07-01T22:30:00Z"
      }
    ],
    "next_cursor": "page_token_string"
  }
  ```

#### 5. Join or Request to Join
* **Endpoint**: `POST /v2/group/{id}/join`
* **Response Status**: `200 OK` (no body)

#### 6. Leave Guild
* **Endpoint**: `POST /v2/group/{id}/leave`
* **Response Status**: `200 OK` (no body)

#### 7. Kick Member
* **Endpoint**: `POST /v2/group/{id}/kick`
* **Request Body**:
  ```json
  {
    "user_ids": ["bc33910c-992e-46aa-b2b9-111122223333"]
  }
  ```
* **Response Status**: `200 OK` (no body)

#### 8. Promote / Demote Member
* **Endpoints**: 
  * `POST /v2/group/{id}/promote`
  * `POST /v2/group/{id}/demote`
* **Request Body**: Same as Kick Member.
* **Response Status**: `200 OK` (no body)

#### 9. List Guild Members
* **Endpoint**: `GET /v2/group/{id}/user`
* **Query Parameters**: Same as normal paginated endpoints (`limit`, `cursor`).
* **Response Body (200 OK)**:
  ```json
  {
    "group_users": [
      {
        "user": {
          "id": "bc33910c-992e-46aa-b2b9-111122223333",
          "username": "shadow_ninja"
        },
        "state": 2 // 0=superadmin, 1=admin, 2=member, 3=join_request
      }
    ],
    "next_cursor": "page_token_string"
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

service GroupService {
  rpc CreateGroup(CreateGroupRequest) returns (Group);
  rpc UpdateGroup(UpdateGroupRequest) returns (google.protobuf.Empty);
  rpc DeleteGroup(DeleteGroupRequest) returns (google.protobuf.Empty);
  rpc ListGroups(ListGroupsRequest) returns (GroupList);
  rpc JoinGroup(JoinGroupRequest) returns (google.protobuf.Empty);
  rpc LeaveGroup(LeaveGroupRequest) returns (google.protobuf.Empty);
  
  rpc AddGroupUsers(AddGroupUsersRequest) returns (google.protobuf.Empty);
  rpc KickGroupUsers(KickGroupUsersRequest) returns (google.protobuf.Empty);
  rpc PromoteGroupUsers(PromoteGroupUsersRequest) returns (google.protobuf.Empty);
  rpc DemoteGroupUsers(DemoteGroupUsersRequest) returns (google.protobuf.Empty);
  rpc BanGroupUsers(BanGroupUsersRequest) returns (google.protobuf.Empty);
  
  rpc ListGroupUsers(ListGroupUsersRequest) returns (GroupUserList);
  rpc ListUserGroups(ListUserGroupsRequest) returns (UserGroupList);
}

message CreateGroupRequest {
  string name = 1;
  string description = 2;
  string avatar_url = 3;
  string lang_tag = 4;
  bool open = 5;
  int32 max_count = 6;
  string metadata = 7;
}

message Group {
  string id = 1;
  string creator_id = 2;
  string name = 3;
  string description = 4;
  string avatar_url = 5;
  string lang_tag = 6;
  bool open = 7;
  int32 edge_count = 8;
  int32 max_count = 9;
  string metadata = 10;
  google.protobuf.Timestamp create_time = 11;
  google.protobuf.Timestamp update_time = 12;
}

message UpdateGroupRequest {
  string id = 1;
  string name = 2;
  string description = 3;
  string avatar_url = 4;
  string lang_tag = 5;
  bool open = 6;
  string metadata = 7;
}

message DeleteGroupRequest {
  string id = 1;
}

message ListGroupsRequest {
  string name = 1;
  string lang_tag = 2;
  bool open = 3;
  int32 limit = 4;
  string cursor = 5;
}

message GroupList {
  repeated Group groups = 1;
  string next_cursor = 2;
}

message JoinGroupRequest {
  string id = 1;
}

message LeaveGroupRequest {
  string id = 1;
}

message AddGroupUsersRequest {
  string group_id = 1;
  repeated string user_ids = 2;
}

message KickGroupUsersRequest {
  string group_id = 1;
  repeated string user_ids = 2;
}

message PromoteGroupUsersRequest {
  string group_id = 1;
  repeated string user_ids = 2;
}

message DemoteGroupUsersRequest {
  string group_id = 1;
  repeated string user_ids = 2;
}

message BanGroupUsersRequest {
  string group_id = 1;
  repeated string user_ids = 2;
}

message ListGroupUsersRequest {
  string group_id = 1;
  int32 limit = 2;
  string cursor = 3;
  int32 state = 4; // 0=superadmin, 1=admin, 2=member, 3=join_request
}

message GroupUser {
  User user = 1;
  int32 state = 2; // 0=superadmin, 1=admin, 2=member, 3=join_request
}

message GroupUserList {
  repeated GroupUser group_users = 1;
  string next_cursor = 2;
}

message ListUserGroupsRequest {
  string user_id = 1;
  int32 limit = 2;
  string cursor = 3;
}

message UserGroup {
  Group group = 1;
  int32 state = 2;
}

message UserGroupList {
  repeated UserGroup user_groups = 1;
  string next_cursor = 2;
}
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `group.max_count` | integer | `100` | Default maximum member count permitted per guild. |
| `group.max_guilds_joined` | integer | `5` | Maximum number of guilds a single player can join. |

---

## Linked Documents
- [BRD-09](../BRD/09_guilds_clans.md) (Business Requirements Document)
- [TDD-09](../TDD/09_guilds_clans.md) (Technical Design Document)
