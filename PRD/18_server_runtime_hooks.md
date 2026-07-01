# PRD-18: Server Runtime & Hooks

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Server Runtime & Hooks  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for the server runtime environment that enables developers to write custom business logic, respond to system events, and extend built-in features. The runtime supports multiple programming languages and provides hooks into all major server events.

---

Refer to [BRD-18](../BRD/18_server_runtime_hooks.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

The runtime intercepts REST, gRPC, and WebSocket traffic. Before hooks run sequentially before built-in logic, while after hooks and event handlers run asynchronously (non-blocking) on separate threads/coroutines.

---

## 3. API Surface Detail

### Initializer Interface
At startup, the server loads custom modules and calls the entry function (`InitModule` in TypeScript, `InitModule` in Go, or top-level script execution in Lua) with an `Initializer` interface.

```typescript
interface Initializer {
  registerRpc(id: string, fn: RpcFunction): void;
  registerMatch(name: string, handler: MatchHandler): void;
  
  // Before Hooks (Intercepts client actions)
  registerBeforeAuthenticateEmail(fn: BeforeHook<AuthenticateEmailRequest>): void;
  registerBeforeWriteStorageObjects(fn: BeforeHook<WriteStorageObjectsRequest>): void;
  registerBeforeAddFriends(fn: BeforeHook<AddFriendsRequest>): void;
  registerBeforeJoinGroup(fn: BeforeHook<JoinGroupRequest>): void;
  
  // After Hooks (Fires after actions execute)
  registerAfterAuthenticateEmail(fn: AfterHook<Session>): void;
  registerAfterWriteStorageObjects(fn: AfterHook<StorageObjectAcks>): void;
  registerAfterAddFriends(fn: AfterHook<empty>): void;
  registerAfterJoinGroup(fn: AfterHook<empty>): void;
  
  // Event Hooks (Asynchronous system events)
  registerEventSessionStart(fn: (ctx: Context, evt: Event) => void): void;
  registerEventSessionEnd(fn: (ctx: Context, evt: Event) => void): void;
  registerEventMatchmakerMatched(fn: (ctx: Context, evt: MatchmakerEvent) => void): void;
  
  // Cron Scheduling
  registerCron(id: string, cronExpr: string, fn: CronFunction): void;
}
```

### Hook Handler Signatures

#### 1. Before Hook Signature
* **Signature**: `beforeHook(ctx, logger, nk, request) -> (request_payload | nil | error)`
* **Behavior**:
  * Return modified request payload to proceed.
  * Return `nil` to proceed with the original request payload.
  * Throw/return an error to reject the request and return the error message to the client (REST returns mapped HTTP error, gRPC returns status code).

#### 2. After Hook Signature
* **Signature**: `afterHook(ctx, logger, nk, response, request)`
* **Behavior**:
  * Executes after the built-in action. Read-only access to response and original request. Return values are ignored.

#### 3. Event Hook Signature
* **Signature**: `eventHook(ctx, logger, nk, event)`
* **Behavior**:
  * Executes asynchronously in the background. Does not block client response.

---

### Global Server Runtime API Namespace (`nk`)
Available in all runtime environments (Lua `nk.*`, TS/JS `nk.*`, Go `nk` plugin helper):

| Category | Available Functions |
|----------|---------------------|
| **Account** | `accountGetId(userId)`, `accountUpdateId(userId, metadata, ...)` |
| **Storage** | `storageRead(readOps)`, `storageWrite(writeOps)`, `storageDelete(deleteOps)` |
| **Wallet** | `walletUpdate(userId, changeset, metadata, updateLedger)` |
| **Leaderboard** | `leaderboardCreate(...)`, `leaderboardRecordWrite(...)`, `leaderboardRecordDelete(...)` |
| **Notification** | `notificationSend(userId, subject, content, code, senderId, persist)` |
| **Match** | `matchCreate(module, params)`, `matchGet(matchId)`, `matchSignal(matchId, data)` |

---

## 4. Product Configurations

Runtime variables and settings are configured in the server config file.

```yaml
runtime:
  execution_timeout_ms: 5000 # Max duration for an RPC or hook execution (5s)
  max_hook_chain_depth: 3    # Limit on recursive hooks
  env:
    log_level: "info"
    custom_system_url: "https://api.external-system.com"
```

---

## 5. Linked Documents
- [BRD-18](../BRD/18_server_runtime_hooks.md) (Business Requirements Document)
- [TDD-18](../TDD/18_server_runtime_hooks.md) (Technical Design Document)
