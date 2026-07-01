# PRD-14: RPC & Custom APIs

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** RPC & Custom APIs  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a Remote Procedure Call (RPC) system that allows developers to create custom backend functions callable from game clients or other server-side code. RPCs extend the server's functionality beyond built-in features, enabling game-specific business logic.

---

Refer to [BRD-14](../BRD/14_rpc_custom_apis.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

RPCs can be invoked via HTTP REST, gRPC, or WebSocket. They support both Authenticated (using standard session Bearer tokens) and Unauthenticated access. Unauthenticated requests require an HTTP query parameter key to verify client origin.

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Authenticated RPC Call
* **Endpoint**: `POST /v2/rpc/{function_id}`
* **Request Headers**:
  * `Authorization`: `Bearer <session_token>`
  * `Content-Type`: `application/json`
* **Request Body**:
  ```json
  {
    "param1": "value1",
    "param2": 123
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "status": "success",
    "result": {
      "reward_claimed": true
    }
  }
  ```

#### 2. Unauthenticated RPC Call
* **Endpoint**: `POST /v2/rpc/{function_id}`
* **Query Parameters**:
  * `http_key` (string, required): Developer-configured client API key.
* **Request Body / Response**: Same as Authenticated RPC Call.

---

### WebSocket Messages

#### 1. Invoke RPC (Client -> Server)
* **Type Name**: `rpc`
* **Payload**:
  ```json
  {
    "id": "ClaimDailyReward",
    "payload": "{\"client_version\":\"1.0.0\"}"
  }
  ```

#### 2. RPC Response (Server -> Client)
* **Type Name**: `rpc`
* **Payload**:
  ```json
  {
    "id": "ClaimDailyReward",
    "payload": "{\"success\":true,\"coins_granted\":100}"
  }
  ```

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

service RpcService {
  rpc RpcFunc(RpcRequest) returns (RpcResponse);
}

message RpcRequest {
  string id = 1;      // The function name
  string payload = 2; // JSON string payload
  string http_key = 3; // Authentication key (for unauthenticated endpoints)
}

message RpcResponse {
  string id = 1;      // The function name
  string payload = 2; // JSON string response payload
}
```

---

### Server Runtime Registration (TypeScript Example)

```typescript
// Define handler logic
function claimDailyReward(ctx: nkruntime.Context, logger: nkruntime.Logger, nk: nkruntime.Nakama, payload: string): string {
  // logic...
  return JSON.stringify({ success: true });
}

// Register handler
initializer.registerRpc("ClaimDailyReward", claimDailyReward);
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `rpc.execution_timeout_ms` | integer | `5000` | Hard timeout for custom script execution (5 seconds). |
| `rpc.max_payload_size_bytes` | integer | `262144` | Maximum size of incoming RPC body (256 KB). |
| `rpc.http_key` | string | `""` | Key required to execute unauthenticated RPCs from game clients. |

---

## Linked Documents
- [BRD-14](../BRD/14_rpc_custom_apis.md) (Business Requirements Document)
- [TDD-14](../TDD/14_rpc_custom_apis.md) (Technical Design Document)
