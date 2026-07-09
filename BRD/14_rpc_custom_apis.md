# BRD-14: RPC & Custom APIs

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** RPC & Custom APIs  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a Remote Procedure Call (RPC) system that allows developers to create custom backend functions callable from game clients or other server-side code. RPCs extend the server's functionality beyond built-in features, enabling game-specific business logic.
---
## 2. Scope

### In Scope
- RPC function registration (Lua, TypeScript, Go)
- Client-to-server RPC calls
- Server-to-server RPC calls
- Authentication and authorization for RPCs
- Request/response payload format
- Rate limiting and throttling
- Error handling and response codes
- HTTP and gRPC RPC invocation

### Out of Scope
- Built-in API endpoints (covered by individual feature BRDs)
- WebSocket message handlers (see BRD-03)
- Match handler functions (see BRD-04)
- Scheduled jobs (see BRD-18: Server Runtime)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Create custom backend logic |
| Players | Access to game-specific features |
| Server Administrators | Monitor and manage custom functions |
| Security Team | Secure execution of custom code |
---
## 4. Functional Requirements

### FR-RPC-001: RPC Registration
- **Priority:** Must
- Developers shall register named RPC functions in the server runtime.
- Registration associates a function name (string) with a handler.
- Function names must be unique.
- Registration happens at server startup (init phase).
- Languages: Lua, TypeScript, Go.
- Go modules are compiled natively as `.so` plugins and loaded via `plugin.Open()`. They run without VM sandboxing and have direct `*sql.DB` database access.
- Lua and TypeScript handlers execute within isolated VM sandboxes.

### FR-RPC-002: RPC Function Signature
- **Priority:** Must
- RPC handlers shall follow a standard signature:
  ```
  function(context, payload) → (response, error)
  ```
- `context`: contains caller info (user_id, username, session vars, client IP, etc.)
- `payload`: string (typically JSON-encoded)
- Returns: string response (typically JSON-encoded) or error
- Go RPC handlers receive an extended signature with direct access to the logger, database, and runtime module:
  ```go
  func(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.Module, payload string) (string, error)
  ```
- For unauthenticated RPCs, context user fields are empty.

### FR-RPC-003: Client-to-Server RPC
- **Priority:** Must
- Clients shall invoke RPCs via:
  - HTTP REST: `POST /v2/rpc/{function_name}`
  - gRPC: `RpcFunc(RpcRequest) returns (RpcResponse)`
  - WebSocket: `rpc` message type
- Request includes:
  - `id`: function name
  - `payload`: string body
- Response includes:
  - `id`: function name
  - `payload`: string response body

### FR-RPC-004: Server-to-Server RPC
- **Priority:** Should
- Server runtime code shall invoke other RPC functions internally.
- Useful for: code reuse, modular logic, cross-feature calls.
- API: `nk.rpc_call(function_name, payload)` → response

### FR-RPC-005: Authentication Modes
- **Priority:** Must
- RPCs shall support two modes:
  - **Authenticated**: requires a valid session token. User context is populated.
  - **Unauthenticated**: callable without authentication (e.g., server status, version check).
- Mode is determined by whether `http_key` is used (unauthenticated) or Bearer token.

### FR-RPC-006: RPC Context
- **Priority:** Must
- The context object shall provide:

| Field | Description |
|-------|-------------|
| `user_id` | Caller's user ID (if authenticated) |
| `username` | Caller's username |
| `vars` | Session variables (custom key-value) |
| `client_ip` | Caller's IP address |
| `client_port` | Caller's port |
| `env` | Server environment variables |
| `execution_mode` | How the RPC was called (REST, gRPC, WebSocket) |
| `headers` | HTTP headers (REST only) |
| `query_params` | URL query parameters (REST only) |

### FR-RPC-007: Error Handling
- **Priority:** Must
- RPC functions shall return structured errors.
- Error response includes:
  - HTTP status code (mapped from runtime error)
  - gRPC status code
  - Error message string
  - Optional error details (JSON)
- Common error codes:

| Code | gRPC | HTTP | Meaning |
|------|------|------|---------|
| NOT_FOUND | 5 | 404 | Resource not found |
| INVALID_ARGUMENT | 3 | 400 | Invalid request |
| PERMISSION_DENIED | 7 | 403 | Not authorized |
| INTERNAL | 13 | 500 | Server error |
| UNAVAILABLE | 14 | 503 | Service unavailable |

### FR-RPC-008: Rate Limiting
- **Priority:** Should
- RPC calls shall be rate-limited per user and per IP.
- Configurable limits per function:
  - Requests per second
  - Requests per minute
  - Burst size
- Exceeded rate returns HTTP 429 / gRPC RESOURCE_EXHAUSTED.

### FR-RPC-009: Payload Size Limits
- **Priority:** Should
- Maximum RPC payload size: configurable (default: 256 KB).
- Maximum response size: configurable (default: 256 KB).
- Exceeding limits returns an error.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | RPC execution overhead ≤ 10ms (excluding handler logic) |
| **Throughput** | Support 50,000+ RPC calls per second per node |
| **Isolation** | RPC errors shall not crash the server |
| **Security** | Authenticated RPCs require valid session token |
| **Monitoring** | Per-function call count, latency, error rate metrics |
| **Logging** | Configurable per-function logging |
---
## 6. Example RPC Functions

### ClaimDailyReward (Lua)
```lua
local function claim_daily_reward(context, payload)
  local user_id = context.user_id
  
  -- Check last claim time from storage
  local reads = {{ collection = "rewards", key = "daily", user_id = user_id }}
  local result = nk.storage_read(reads)
  
  local now = os.time()
  if result[1] then
    local data = result[1].value
    if now - data.last_claim < 86400 then
      error({ code = 3, message = "Already claimed today" })
    end
  end
  
  -- Grant reward
  local changeset = { coins = 100, gems = 5 }
  nk.wallet_update(user_id, changeset, { reason = "daily_reward" })
  
  -- Update claim record
  local writes = {{
    collection = "rewards",
    key = "daily",
    user_id = user_id,
    value = { last_claim = now, streak = (result[1] and result[1].value.streak or 0) + 1 }
  }}
  nk.storage_write(writes)
  
  return nk.json_encode({ reward = changeset })
end

nk.register_rpc(claim_daily_reward, "ClaimDailyReward")
```

### BuyItem (TypeScript)
```typescript
const buyItem: nkruntime.RpcFunction = function(ctx, logger, nk, payload) {
  const request = JSON.parse(payload);
  const { item_id, quantity } = request;
  
  // Look up item in catalog
  const catalog = nk.storageRead([{
    collection: "catalog", key: item_id, userId: null
  }]);
  if (catalog.length === 0) {
    throw Error("Item not found");
  }
  
  const item = catalog[0].value;
  const totalCost = { coins: -(item.price.coins * quantity) };
  
  // Debit wallet (throws if insufficient)
  nk.walletUpdate(ctx.userId, totalCost, { reason: "purchase", item_id });
  
  // Add to inventory
  // ... storage write logic
  
  return JSON.stringify({ success: true, item_id, quantity });
};

initializer.registerRpc("BuyItem", buyItem);
```

### ClaimDailyReward (Go)
```go
package main

import (
	"context"
	"database/sql"
	"encoding/json"

	"github.com/heroiclabs/nakama-common/runtime"
)

func ClaimDailyReward(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.Module, payload string) (string, error) {
	userID, ok := ctx.Value(runtime.RUNTIME_CTX_USER_ID).(string)
	if !ok {
		return "", runtime.NewError("missing user id", 3) // INVALID_ARGUMENT
	}

	// Read last claim time from storage
	reads := []*runtime.StorageRead{{
		Collection: "rewards",
		Key:        "daily",
		UserID:     userID,
	}}
	records, err := nk.StorageRead(ctx, reads)
	if err != nil {
		return "", runtime.NewError("storage read failed", 13) // INTERNAL
	}

	// Grant reward via wallet update
	changeset := map[string]int64{"coins": 100, "gems": 5}
	metadata := map[string]interface{}{"reason": "daily_reward"}
	if err := nk.WalletUpdate(ctx, userID, changeset, metadata, true); err != nil {
		return "", runtime.NewError("wallet update failed", 13)
	}

	response, _ := json.Marshal(map[string]interface{}{"reward": changeset})
	return string(response), nil
}

// InitModule registers all Go RPCs at server startup.
func InitModule(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.Module, initializer runtime.Initializer) error {
	if err := initializer.RegisterRpc("ClaimDailyReward", ClaimDailyReward); err != nil {
		return err
	}
	logger.Info("Go module loaded successfully")
	return nil
}
```
---
## 7. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Session validation for authenticated RPCs |
| BRD-18: Server Runtime & Hooks | Execution environment for RPC handlers |
| BRD-12: Storage Engine | Data access within RPCs |
| BRD-13: Economy System | Wallet operations within RPCs |
| BRD-05: Leaderboards | Score submission within RPCs |
---
## 8. Acceptance Criteria

- [ ] RPC functions can be registered in Lua, TypeScript, and Go
- [ ] Clients can call RPCs via REST, gRPC, and WebSocket
- [ ] Authenticated RPCs require valid session token
- [ ] Unauthenticated RPCs work with HTTP key
- [ ] Context provides caller identity, IP, and environment
- [ ] Errors are returned with appropriate status codes
- [ ] Rate limiting prevents RPC abuse
- [ ] RPC errors do not crash the server
- [ ] Per-function metrics are available
---
## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| RPC handler infinite loop | Server thread starvation | Medium | Execution timeout per RPC call |
| Unvalidated input in RPCs | Security vulnerabilities | High | Developer guidance; input validation best practices |
| RPC abuse (unauthenticated) | DDoS vector | Medium | HTTP key rotation; rate limiting; IP filtering |
| Large payload attacks | Memory exhaustion | Low | Payload size limits |
| Poor error handling in RPCs | Leaked internal data | Medium | Error sanitization; structured error framework |
---
## 10. Future Considerations

- GraphQL API layer on top of RPCs
- RPC versioning (v1, v2 of same function)
- Async RPCs (fire-and-forget with callback)
- RPC marketplace (shared community functions)
- Automatic API documentation generation from RPC registrations
- RPC testing framework (mock context, assertions)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |
| 1.1 | 2026-07-09 | Engineering | Added Go native runtime RPC example, clarified Go execution model (non-sandboxed, direct DB access) |

---

## Linked Documents
- [PRD-14](../PRD/14_rpc_custom_apis.md) (API Surface & Interface Specification)
- [TDD-14](../TDD/14_rpc_custom_apis.md) (Database Schema & Technical Implementation Design)
