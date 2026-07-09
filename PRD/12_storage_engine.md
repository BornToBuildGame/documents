# PRD-12: Storage Engine

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Storage Engine  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a NoSQL-style document storage layer that enables developers to store and retrieve arbitrary JSON data per user or globally. This system stores player inventory, settings, quest progress, character data, cosmetics, and any game-specific data.

---

Refer to [BRD-12](../BRD/12_storage_engine.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Storage engine operations are exposed over REST and gRPC. To guarantee security, client requests are evaluated against read/write permission scopes, whereas server-side runtime scripts bypass all restrictions.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Write Storage Objects (Batch)
* **Endpoint**: `POST /v2/storage`
* **Request Body**:
  ```json
  {
    "objects": [
      {
        "collection": "inventory",
        "key": "weapons",
        "value": {
          "swords": ["excalibur", "katana"],
          "shields": ["wooden"]
        },
        "version": "",
        "permission_read": 1,
        "permission_write": 1
      }
    ]
  }
  ```
  *(Note: `version` can be provided for optimistic concurrency/conditional writes. Provide a specific version hash to match the existing record version, or `"*"` to specify that the write should succeed only if the object does not exist yet).*
* **Response Body (200 OK)**:
  ```json
  {
    "acks": [
      {
        "collection": "inventory",
        "key": "weapons",
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "version": "cf84d9f123abc456",
        "create_time": "2026-07-01T22:30:00Z",
        "update_time": "2026-07-01T22:30:00Z"
      }
    ]
  }
  ```

#### 2. Read Storage Objects (Batch)
* **Endpoint**: `POST /v2/storage/read`
* **Request Body**:
  ```json
  {
    "object_ids": [
      {
        "collection": "inventory",
        "key": "weapons",
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0"
      }
    ]
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "objects": [
      {
        "collection": "inventory",
        "key": "weapons",
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "value": {
          "swords": ["excalibur", "katana"],
          "shields": ["wooden"]
        },
        "version": "cf84d9f123abc456",
        "permission_read": 1,
        "permission_write": 1,
        "create_time": "2026-07-01T22:30:00Z",
        "update_time": "2026-07-01T22:30:00Z"
      }
    ]
  }
  ```

#### 3. Delete Storage Objects (Batch)
* **Endpoint**: `POST /v2/storage/delete`
* **Request Body**:
  ```json
  {
    "object_ids": [
      {
        "collection": "inventory",
        "key": "weapons",
        "version": "cf84d9f123abc456"
      }
    ]
  }
  ```
* **Response Status**: `200 OK` (no body)

#### 4. List Storage Objects
* **Endpoint**: `GET /v2/storage/{collection}`
* **Query Parameters**:
  * `user_id` (string, optional): Owner filter. If empty or `""`, public records in the collection across all users (where `permission_read = 2`) are queried.
  * `limit` (integer, optional, default: `20`, max: `100`)
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**: Same structure as Read Storage Objects, including `next_cursor` pagination field.

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";

service StorageService {
  rpc ReadStorageObjects(ReadStorageObjectsRequest) returns (StorageObjects);
  rpc WriteStorageObjects(WriteStorageObjectsRequest) returns (StorageObjectAcks);
  rpc DeleteStorageObjects(DeleteStorageObjectsRequest) returns (google.protobuf.Empty);
  rpc ListStorageObjects(ListStorageObjectsRequest) returns (StorageObjectList);
}

message ReadStorageObjectsRequest {
  message ReadOp {
    string collection = 1;
    string key = 2;
    string user_id = 3;
  }
  repeated ReadOp object_ids = 1;
}

message StorageObject {
  string collection = 1;
  string key = 2;
  string user_id = 3;
  string value = 4; // JSON string format
  string version = 5;
  int32 permission_read = 6;
  int32 permission_write = 7;
  google.protobuf.Timestamp create_time = 8;
  google.protobuf.Timestamp update_time = 9;
}

message StorageObjects {
  repeated StorageObject objects = 1;
}

message WriteStorageObjectsRequest {
  message WriteOp {
    string collection = 1;
    string key = 2;
    string value = 3; // JSON string format
    string version = 4;
    int32 permission_read = 5;
    int32 permission_write = 6;
  }
  repeated WriteOp objects = 1;
}

message StorageObjectAck {
  string collection = 1;
  string key = 2;
  string user_id = 3;
  string version = 4;
  google.protobuf.Timestamp create_time = 5;
  google.protobuf.Timestamp update_time = 6;
}

message StorageObjectAcks {
  repeated StorageObjectAck acks = 1;
}

message DeleteStorageObjectsRequest {
  message DeleteOp {
    string collection = 1;
    string key = 2;
    string version = 3;
  }
  repeated DeleteOp object_ids = 1;
}

message ListStorageObjectsRequest {
  string collection = 1;
  string user_id = 2;
  int32 limit = 3;
  string cursor = 4;
}

message StorageObjectList {
  repeated StorageObject objects = 1;
  string next_cursor = 2;
}
```

---

### Server Runtime API

```lua
nk.storage_read(object_ids)
nk.storage_write(objects)
nk.storage_delete(object_ids)
nk.storage_list(user_id, collection, limit, cursor)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `storage.max_object_size_bytes` | integer | `262144` | Maximum size of JSON document (256 KB). |
| `storage.max_objects_per_batch` | integer | `128` | Maximum number of objects processed in a single batch read/write. |

---

## Linked Documents
- [BRD-12](../BRD/12_storage_engine.md) (Business Requirements Document)
- [TDD-12](../TDD/12_storage_engine.md) (Technical Design Document)
