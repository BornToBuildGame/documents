# PRD-19: API Layer (REST, gRPC, WebSocket)

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** API Layer  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for the server's communication layer supporting HTTP REST, WebSocket, and gRPC protocols. The API layer serves as the gateway for all client-server and server-to-server interactions, providing consistent, secure, and performant access to all server features.

---

Refer to [BRD-19](../BRD/19_api_layer.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

The API gateway exposes three ports for external access:
* **HTTP REST & WebSocket**: Default port `7350`
* **gRPC**: Default port `7349`
* **Developer Admin Console (Web Dashboard)**: Default port `7351`

Production traffic must employ TLS 1.3 for encryption.

---

## 3. API Surface Detail

### REST Endpoint Conventions
All REST endpoints are path-versioned under `/v2/`. Request payloads are JSON-formatted. Error responses are formatted according to the common schema outlined in [PRD-01](./01_user_authentication.md).

#### 1. System Health Check
* **Endpoint**: `GET /healthcheck`
* **Authentication**: None
* **Response Body (200 OK)**:
  ```json
  {
    "status": "ok",
    "version": "1.0.0",
    "timestamp": "2026-07-01T22:30:00Z"
  }
  ```

---

### WebSocket Envelope Structure
WebSocket connections represent a multiplexed bi-directional stream. All socket frames use a standardized JSON wrapper containing a `cid` (correlation ID) to match responses to requests.

#### Client-to-Server Wrapper:
```json
{
  "cid": "client-request-id-1",
  "matchmaker_add": {
    "queue_name": "casual_5v5"
  }
}
```

#### Server-to-Client Wrapper (Replies):
```json
{
  "cid": "client-request-id-1",
  "matchmaker_ticket": {
    "ticket_id": "ticket-uuid-2222"
  }
}
```

#### Server-to-Client Wrapper (Events/Pushes):
If the message is an asynchronous server push, the `cid` field is omitted.
```json
{
  "notifications": [
    {
      "id": "notification-uuid-9999",
      "subject": "Friend Request"
    }
  ]
}
```

---

### Global HTTP Headers

#### Security Headers (REST Responses):
* `X-Content-Type-Options`: `nosniff`
* `X-Frame-Options`: `DENY`
* `X-Request-Id`: `<trace-uuid>`

#### Rate Limiting Headers:
When rate-limiting is active, all HTTP endpoints return:
* `X-RateLimit-Limit`: Maximum requests allowed per window.
* `X-RateLimit-Remaining`: Requests remaining in current window.
* `X-RateLimit-Reset`: UNIX timestamp when current window resets.

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `api.port` | integer | `7350` | Port listening for HTTP/REST and WebSocket traffic. |
| `api.grpc_port` | integer | `7349` | Port listening for gRPC requests. |
| `api.console_port` | integer | `7351` | Port hosting the developer console Web UI dashboard. |
| `api.cors.allowed_origins` | array of strings | `["*"]` | Permitted cross-origin resource sharing targets. |
| `api.rate_limit.max_requests_per_sec` | integer | `100` | Rate limit threshold per IP/User (100 req/sec). |

---

## Linked Documents
- [BRD-19](../BRD/19_api_layer.md) (Business Requirements Document)
- [TDD-19](../TDD/19_api_layer.md) (Technical Design Document)
