# BRD-19: API Layer (REST, gRPC, WebSocket)

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** API Layer  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for the server's communication layer supporting HTTP REST, WebSocket, and gRPC protocols. The API layer serves as the gateway for all client-server and server-to-server interactions, providing consistent, secure, and performant access to all server features.
---
## 2. Scope

### In Scope
- HTTP REST API
- WebSocket API (real-time)
- gRPC API (server-to-server, high-performance)
- API versioning
- Authentication and authorization middleware
- Request validation
- Rate limiting
- CORS and security headers
- SDK generation and client library support
- API documentation

### Out of Scope
- GraphQL (future consideration)
- Specific endpoint implementations (covered by feature BRDs)
- Client SDK implementation details
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Clear APIs, SDK availability, documentation |
| Client SDK Teams | Consistent API contracts |
| Server Engineers | Performance, security, scalability |
| DevOps | Monitoring, rate limiting, deployment |
---
## 4. Functional Requirements

### FR-API-001: HTTP REST API
- **Priority:** Must
- RESTful API for request/response operations.
- Base URL: `/v2/` (versioned)
- Content type: `application/json`
- Authentication: `Authorization: Bearer {session_token}`
- Unauthenticated endpoints use: `?http_key={server_key}`
- Standard HTTP methods: GET, POST, PUT, DELETE
- Response format:
  ```json
  {
    "field": "value",
    "create_time": "2026-07-01T00:00:00Z"
  }
  ```
- Error response format:
  ```json
  {
    "error": "Error message",
    "code": 3,
    "message": "Detailed error description"
  }
  ```

### FR-API-002: WebSocket API
- **Priority:** Must
- Persistent WebSocket connection for real-time features.
- Connection URL: `ws://host:port/ws?token={session_token}`
- Message format: JSON (or binary Protobuf, configurable)
- Message envelope:
  ```json
  {
    "cid": "1",
    "<message_type>": {
      ... message-specific fields
    }
  }
  ```
- `cid` (correlation ID): used to match responses to requests.
- Server pushes (no cid): notifications, presence, match data.
- Supported message types: all real-time operations (match, party, chat, status, matchmaker, rpc).

### FR-API-003: gRPC API
- **Priority:** Must
- gRPC service for server-to-server and high-performance client communication.
- Protocol: HTTP/2 + Protocol Buffers
- Service definition from `.proto` files.
- Authentication: metadata header `authorization: Bearer {token}`
- Supports all API operations available via REST.
- Streaming RPCs for real-time features (bi-directional streaming).

### FR-API-004: API Versioning
- **Priority:** Must
- API version prefix in URL path: `/v2/...`
- Version in gRPC package name.
- Breaking changes require new major version.
- Deprecated endpoints marked with response headers.
- Minimum supported version: configurable.

### FR-API-005: Authentication Middleware
- **Priority:** Must
- All protected endpoints require valid session token.
- Token validation: signature check, expiry check, user lookup.
- Middleware extracts user context for downstream handlers.
- Server key (`http_key`) for unauthenticated access.
- gRPC interceptor for auth validation.

### FR-API-006: Request Validation
- **Priority:** Must
- All incoming requests shall be validated:
  - Required fields present
  - Field types correct
  - String lengths within limits
  - Enum values valid
  - UUIDs properly formatted
- Validation errors return HTTP 400 / gRPC INVALID_ARGUMENT.

### FR-API-007: Rate Limiting
- **Priority:** Must
- Global and per-endpoint rate limiting.
- Limits configurable:
  - Per IP
  - Per user
  - Per endpoint
- Rate limit headers in REST responses:
  - `X-RateLimit-Limit`
  - `X-RateLimit-Remaining`
  - `X-RateLimit-Reset`
- Exceeded rate: HTTP 429 / gRPC RESOURCE_EXHAUSTED.

### FR-API-008: CORS Configuration
- **Priority:** Should
- Configurable CORS headers for browser-based clients.
- Settings:
  - `allowed_origins`: list of allowed origins (or `*`)
  - `allowed_methods`: allowed HTTP methods
  - `allowed_headers`: allowed request headers
  - `max_age`: preflight cache duration
- Default: permissive for development, restricted for production.

### FR-API-009: Security Headers
- **Priority:** Should
- Standard security headers on all responses:
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
  - `Strict-Transport-Security` (when TLS enabled)
  - `X-Request-Id` (correlation ID for tracing)

### FR-API-010: TLS/SSL Support
- **Priority:** Must
- Support TLS for all protocols:
  - HTTPS (REST)
  - WSS (WebSocket Secure)
  - gRPC with TLS
- Certificate configuration: path to cert and key files.
- Support for Let's Encrypt auto-renewal (optional).

### FR-API-011: Request/Response Logging
- **Priority:** Should
- Configurable request logging:
  - Endpoint, method, status code, duration
  - Client IP, user_id
  - Request/response body (in debug mode only)
- Log format: structured JSON.
- Sensitive data (tokens, passwords) shall never be logged.

### FR-API-012: Health Check Endpoint
- **Priority:** Must
- Unprotected health check: `GET /healthcheck`
- Returns: HTTP 200 with server status.
- Response:
  ```json
  {
    "status": "ok",
    "version": "1.0.0",
    "timestamp": "2026-07-01T00:00:00Z"
  }
  ```
- Used by load balancers and monitoring.

### FR-API-013: Client SDKs
- **Priority:** Should
- Provide client SDKs for a broad range of game engines, frameworks, and programming languages:

| Engine / Platform | Language / SDK Type |
|-------------------|---------------------|
| **Unity** | C# / .NET |
| **Unreal Engine** | C++ |
| **Godot 3 & 4** | GDScript / C# |
| **MonoGame** | C# / .NET |
| **LibGDX** | Java |
| **Defold** | Lua |
| **Cocos2d-x (C++ & JS)** | C++ / JavaScript |
| **Phaser** | JavaScript |
| **Macroquad** | Rust |
| **iOS / macOS** | Swift |
| **Android** | Java / Kotlin |
| **Flutter** | Dart |
| **General Web** | JavaScript / TypeScript |

- SDKs shall be generated or adapted from API specifications (OpenAPI / Protobuf).
- SDKs handle: authentication, session token refresh, WebSocket socket lifecycle management, connection retry, serialization, and real-time event routing.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | API overhead ≤ 10ms (excluding handler logic) |
| **Throughput** | Support 100,000+ HTTP requests/second per node |
| **WebSocket** | Support 100,000+ concurrent connections per node |
| **gRPC** | Support 50,000+ RPC calls/second per node |
| **TLS** | TLS 1.2+ required for production |
| **Availability** | 99.9% uptime |
| **Monitoring** | Request count, latency, error rate, connection count |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Token validation middleware |
| All feature BRDs | Each feature exposes API endpoints |
| Protocol Buffers | gRPC service definitions |
| OpenAPI / Swagger | REST API documentation |
---
## 7. Acceptance Criteria

- [ ] REST API serves all endpoints on `/v2/` path
- [ ] WebSocket connections are established with session token
- [ ] gRPC service handles all operations available via REST
- [ ] Authentication middleware validates tokens on all protected endpoints
- [ ] Rate limiting enforces configured limits per IP and per user
- [ ] TLS/SSL is configurable for all protocols
- [ ] Health check endpoint responds without authentication
- [ ] CORS headers are configurable
- [ ] Request logging captures method, endpoint, status, duration
- [ ] API responds with structured error format
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| DDoS attacks | Service unavailability | Medium | Rate limiting; CDN; IP blocking |
| WebSocket connection exhaustion | Server overload | Medium | Connection limits; backpressure |
| TLS certificate expiry | Service disruption | Low | Auto-renewal; monitoring alerts |
| API breaking changes | Client compatibility | Medium | Strict versioning; deprecation warnings |
| SDK version fragmentation | Support burden | Medium | SDK compatibility matrix; forced upgrades |
---
## 9. Future Considerations

- GraphQL API layer
- HTTP/3 (QUIC) support
- WebTransport for unreliable data channels
- API Gateway (external, e.g., Kong, Envoy)
- API analytics and usage dashboards
- OpenAPI 3.1 auto-generation from code
- SDK auto-generation pipeline
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-19](../PRD/19_api_layer.md) (API Surface & Interface Specification)
- [TDD-19](../TDD/19_api_layer.md) (Database Schema & Technical Implementation Design)
