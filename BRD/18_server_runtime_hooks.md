# BRD-18: Server Runtime & Hooks

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Server Runtime & Hooks  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for the server runtime environment that enables developers to write custom business logic, respond to system events, and extend built-in features. The runtime supports multiple programming languages and provides hooks into all major server events.
---
## 2. Scope

### In Scope
- Server runtime execution environment
- Before/after hooks on built-in APIs
- Event hooks (login, match, storage, etc.)
- Scheduled jobs (cron-based)
- Multi-language support (Lua, TypeScript, Go)
- Server runtime API (access to all server features)
- Initialization and module loading

### Out of Scope
- Match handler logic (see BRD-04, which runs within the runtime)
- RPC function definitions (see BRD-14, which registers within the runtime)
- Runtime deployment / CI/CD
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Write server-side game logic |
| Server Administrators | Deploy and manage server code |
| DevOps | Runtime configuration, monitoring |
| Security Team | Code execution sandboxing |
---
## 4. Functional Requirements

### FR-SR-001: Runtime Languages
- **Priority:** Must
- The server runtime shall support three languages:

| Language | Execution Model | Hot Reload | Use Case |
|----------|----------------|------------|----------|
| **Lua** | Embedded VM (LuaJIT) | Yes | Rapid prototyping, lightweight logic |
| **TypeScript** | Compiled to JS, embedded runtime | Yes | Full-featured game logic |
| **Go** | Compiled plugin or built-in | No (restart) | High-performance, production |

- All languages shall have access to the same server API surface.
- Language-specific modules are loaded at startup.

### FR-SR-002: Initialization
- **Priority:** Must
- The runtime shall execute initialization code at server startup.
- Init functions register:
  - RPC handlers
  - Match handlers
  - Before/after hooks
  - Event listeners
  - Scheduled jobs
- Init receives a context with server configuration.

### FR-SR-003: Before Hooks
- **Priority:** Must
- Developers shall register "before" hooks on built-in API endpoints.
- Before hooks execute before the built-in handler processes the request.
- Before hooks can:
  - Modify the request
  - Reject the request (return error)
  - Allow the request to proceed (pass through)
- Available for all API endpoints:

| Category | Hookable Endpoints |
|----------|-------------------|
| Auth | authenticate_email, authenticate_device, authenticate_* |
| Social | add_friends, block_friends, delete_friends |
| Groups | join_group, leave_group, update_group, promote, kick |
| Chat | channel_message_send |
| Leaderboards | write_leaderboard_record |
| Tournaments | join_tournament, write_tournament_record |
| Storage | write_storage_objects, read_storage_objects, delete_storage_objects |
| Matchmaker | matchmaker_add, matchmaker_remove |
| Account | update_account, get_account |

### FR-SR-004: After Hooks
- **Priority:** Must
- Developers shall register "after" hooks on built-in API endpoints.
- After hooks execute after the built-in handler has processed the request.
- After hooks receive both the request and the response/result.
- After hooks can:
  - Trigger side effects (notifications, analytics, achievements)
  - Log activity
  - Modify the response (limited)
- Cannot reject the request (already processed).

### FR-SR-005: Event Hooks
- **Priority:** Must
- The runtime shall emit events for major server actions:

| Event | Trigger |
|-------|---------|
| `session_start` | User connects |
| `session_end` | User disconnects |
| `matchmaker_matched` | Matchmaker forms a match |
| `leaderboard_reset` | Leaderboard resets |
| `tournament_end` | Tournament ends |
| `tournament_reset` | Tournament resets |
| `purchase_validation` | IAP receipt submitted |

- Event handlers are registered during initialization.
- Events are processed asynchronously (non-blocking).

### FR-SR-006: Scheduled Jobs
- **Priority:** Should
- The runtime shall support scheduled jobs via cron expressions.
- Job registration: function + cron expression.
- Jobs execute server-side at the specified schedule.
- Use cases:
  - Daily reward reset
  - Periodic cleanup
  - Event rotation
  - Analytics aggregation
  - Maintenance tasks
- Jobs shall be distributed across cluster nodes (one execution per schedule).

### FR-SR-007: Server Runtime API
- **Priority:** Must
- The runtime API (`nk` namespace) shall provide access to:

| Category | Functions |
|----------|----------|
| **Users** | account_get, account_update, users_get, users_ban, users_unban |
| **Auth** | authenticate_*, link_*, unlink_* |
| **Storage** | storage_read, storage_write, storage_delete, storage_list |
| **Wallet** | wallet_update, wallets_update, wallet_ledger_list |
| **Leaderboards** | leaderboard_create, leaderboard_record_write, leaderboard_records_list |
| **Tournaments** | tournament_create, tournament_join, tournament_record_write |
| **Friends** | friends_add, friends_delete, friends_block, friends_list |
| **Groups** | group_create, group_update, group_delete, group_users_list |
| **Chat** | channel_message_send, channel_message_list |
| **Notifications** | notification_send, notifications_send |
| **Matches** | match_create, match_list, match_get, match_signal |
| **Matchmaker** | matchmaker_add |
| **Streams** | stream_user_list, stream_user_join, stream_user_leave, stream_send |
| **Utilities** | json_encode, json_decode, uuid_v4, time, logger, http_request, base64, crypto |

### FR-SR-008: Logging
- **Priority:** Must
- Runtime code shall have access to a structured logger.
- Log levels: debug, info, warn, error.
- Logs shall include context (function name, user_id, match_id).
- Configurable log level per module.

### FR-SR-009: HTTP Requests
- **Priority:** Should
- Server runtime code shall make outbound HTTP requests.
- Used for: external API calls, webhooks, analytics.
- Request options: method, URL, headers, body, timeout.
- Response includes: status code, headers, body.
- Configurable: timeout, max redirects, TLS verification.

### FR-SR-010: Error Handling
- **Priority:** Must
- Runtime errors shall be caught and handled gracefully.
- Errors in hooks shall not crash the server.
- Errors are logged with stack traces (in debug mode).
- Runtime panics in Go plugins shall be recovered.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Hook execution overhead ≤ 5ms |
| **Isolation** | Runtime errors do not crash the server |
| **Hot Reload** | Lua/TypeScript support reload without restart |
| **Security** | Sandboxed execution (no filesystem access from Lua/TS) |
| **Scalability** | Runtime scales with server instances |
| **Monitoring** | Hook execution time, error rates, scheduled job status |
---
## 6. Example: Initialization (TypeScript)

```typescript
function InitModule(
  ctx: nkruntime.Context,
  logger: nkruntime.Logger,
  nk: nkruntime.Nakama,
  initializer: nkruntime.Initializer
) {
  // Register RPC functions
  initializer.registerRpc("ClaimDailyReward", claimDailyReward);
  initializer.registerRpc("BuyItem", buyItem);
  
  // Register match handler
  initializer.registerMatch("battle_royale", {
    matchInit, matchJoinAttempt, matchJoin, matchLeave, matchLoop, matchTerminate
  });
  
  // Register before/after hooks
  initializer.registerBeforeAuthenticateEmail(beforeAuthEmail);
  initializer.registerAfterWriteStorageObjects(afterStorageWrite);
  
  // Register event handler
  initializer.registerEvent(function(ctx, evt) {
    logger.info("Event: %s", evt.name);
  });
  
  // Register scheduled job
  initializer.registerCron("resetDailyQuests", "0 0 * * *", resetDailyQuests);
  
  logger.info("Server runtime initialized.");
}
```
---
## 7. Dependencies

| Dependency | Purpose |
|------------|---------|
| All feature BRDs | Runtime provides hooks and API access for all features |
| Lua VM (LuaJIT) | Lua runtime execution |
| V8 / QuickJS | TypeScript/JavaScript execution |
| Go Plugin System | Go plugin loading |
---
## 8. Acceptance Criteria

- [ ] Server runtime supports Lua, TypeScript, and Go
- [ ] Initialization registers RPC handlers, match handlers, and hooks
- [ ] Before hooks can modify or reject API requests
- [ ] After hooks can trigger side effects
- [ ] Event hooks fire for session, matchmaker, leaderboard, and tournament events
- [ ] Scheduled jobs execute on cron schedule
- [ ] Runtime API provides access to all server features
- [ ] Hot reload works for Lua and TypeScript without server restart
- [ ] Runtime errors are isolated and do not crash the server
- [ ] Structured logging is available in all runtime languages
---
## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Infinite loop in runtime code | Thread starvation | Medium | Execution timeouts; resource limits |
| Memory leak in Lua/JS VM | Server degradation | Medium | VM recycling; memory limits per context |
| Go plugin compatibility | Build issues | Low | Pin Go version; static compilation option |
| Hook chain performance | Request latency | Medium | Limit hook chain depth; async processing |
| Security (outbound HTTP from runtime) | Data exfiltration | Low | URL allowlist; network policy |
---
## 10. Future Considerations

- WebAssembly (Wasm) runtime support
- Runtime module marketplace
- Visual scripting / node-based logic editor
- Runtime debugging (attach debugger, breakpoints)
- Runtime profiling and performance analysis
- Multi-version runtime (A/B test server logic)
- Dependency management for runtime modules
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-18](../PRD/18_server_runtime_hooks.md) (API Surface & Interface Specification)
- [TDD-18](../TDD/18_server_runtime_hooks.md) (Database Schema & Technical Implementation Design)
