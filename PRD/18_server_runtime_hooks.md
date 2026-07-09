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

### Go Initializer Interface

Go modules implement `InitModule` and receive an `Initializer` that provides registration methods. All methods return `error` (unlike TypeScript which returns `void`).

```go
// Initializer provides registration methods available during InitModule execution.
type Initializer interface {
	// Core registrations
	RegisterRpc(id string, fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, payload string) (string, error)) error
	RegisterMatch(name string, fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module) (Match, error)) error

	// Before hooks — intercept and modify/reject client requests
	RegisterBeforeAuthenticateEmail(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, in *AuthenticateEmailRequest) (*AuthenticateEmailRequest, error)) error
	RegisterBeforeWriteStorageObjects(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, in *WriteStorageObjectsRequest) (*WriteStorageObjectsRequest, error)) error
	RegisterBeforeAddFriends(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, in *AddFriendsRequest) (*AddFriendsRequest, error)) error
	RegisterBeforeJoinGroup(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, in *JoinGroupRequest) (*JoinGroupRequest, error)) error

	// After hooks — trigger side effects post-processing
	RegisterAfterAuthenticateEmail(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, out *Session, in *AuthenticateEmailRequest) error) error
	RegisterAfterWriteStorageObjects(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, out *StorageObjectAcks, in *WriteStorageObjectsRequest) error) error
	RegisterAfterAddFriends(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, in *AddFriendsRequest) error) error
	RegisterAfterJoinGroup(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, in *JoinGroupRequest) error) error

	// Event hooks — asynchronous system events
	RegisterEvent(fn func(ctx context.Context, logger Logger, evt *Event)) error
	RegisterEventSessionStart(fn func(ctx context.Context, logger Logger, evt *Event)) error
	RegisterEventSessionEnd(fn func(ctx context.Context, logger Logger, evt *Event)) error
	RegisterMatchmakerMatched(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, entries []MatchmakerEntry) (string, error)) error
	RegisterLeaderboardReset(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, leaderboard *Leaderboard, reset int64) error) error
	RegisterTournamentEnd(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, tournament *Tournament, end int64, reset int64) error) error
	RegisterTournamentReset(fn func(ctx context.Context, logger Logger, db *sql.DB, nk Module, tournament *Tournament, end int64, reset int64) error) error
}
```

### Hook Handler Signatures

#### 1. Before Hook Signature
* **Signature (TypeScript)**: `beforeHook(ctx, logger, nk, request) -> (request_payload | nil | error)`
* **Signature (Go)**: `func(ctx context.Context, logger Logger, db *sql.DB, nk Module, in *Request) (*Request, error)`
* **Behavior**:
  * Return modified request payload to proceed.
  * Return `nil` to proceed with the original request payload.
  * Throw/return an error to reject the request and return the error message to the client (REST returns mapped HTTP error, gRPC returns status code).

#### 2. After Hook Signature
* **Signature (TypeScript)**: `afterHook(ctx, logger, nk, response, request)`
* **Signature (Go)**: `func(ctx context.Context, logger Logger, db *sql.DB, nk Module, out *Response, in *Request) error`
* **Behavior**:
  * Executes after the built-in action. Read-only access to response and original request. Return values are ignored.

#### 3. Event Hook Signature
* **Signature (TypeScript)**: `eventHook(ctx, logger, nk, event)`
* **Signature (Go)**: `func(ctx context.Context, logger Logger, evt *Event)`
* **Behavior**:
  * Executes asynchronously in the background. Does not block client response.

---

### Global Server Runtime API Namespace (`nk`)
Available in all runtime environments (Lua `nk.*`, TS/JS `nk.*`, Go `nk` plugin helper):

| Category | Available Functions / Methods |
|----------|---------------------|
| **Account & Users** | `accountGetId(userId)`, `accountsGetId(userIds)`, `accountUpdateId(userId, metadata, ...)`, `accountDeleteId(userId, record)`, `usersGetId(userIds)`, `usersGetUsername(usernames)`, `usersBanId(userIds)`, `usersUnbanId(userIds)` |
| **Storage Engine** | `storageRead(readOps)`, `storageWrite(writeOps)`, `storageDelete(deleteOps)`, `storageList(callerId, userId, collection, limit, cursor)`, `storageIndexList(callerId, indexName, query, limit, order, cursor)` |
| **Wallet & Economy** | `walletUpdate(userId, changeset, metadata, updateLedger)`, `walletsUpdate(updates, updateLedger)`, `walletLedgerUpdate(itemId, metadata)`, `walletLedgerList(userId, limit, cursor)` |
| **Leaderboards** | `leaderboardCreate(id, authoritative, sort, operator, schedule, metadata, ranks)`, `leaderboardDelete(id)`, `leaderboardList(limit, cursor)`, `leaderboardRecordsList(id, ownerIds, limit, cursor, expiry)`, `leaderboardRecordWrite(id, ownerId, username, score, subscore, metadata, operator)`, `leaderboardRecordDelete(id, ownerId)` |
| **Tournaments** | `tournamentCreate(...)`, `tournamentDelete(id)`, `tournamentAddAttempt(id, ownerId, count)`, `tournamentJoin(id, ownerId, username)`, `tournamentList(...)`, `tournamentRecordsList(...)`, `tournamentRecordWrite(...)`, `tournamentRecordDelete(...)` |
| **Groups & Guilds** | `groupsGetId(groupIds)`, `groupCreate(...)`, `groupUpdate(...)`, `groupDelete(id)`, `groupUserJoin(...)`, `groupUserLeave(...)`, `groupUsersAdd(...)`, `groupUsersKick(...)`, `groupUsersPromote(...)`, `groupUsersDemote(...)`, `groupUsersList(id, limit, state, cursor)`, `groupsList(...)` |
| **Social & Friends** | `friendsList(userId, limit, state, cursor)`, `friendsAdd(userId, username)`, `friendsDelete(userId, username)`, `friendsBlock(userId, username)` |
| **In-App Purchases** | `purchaseValidateApple(userId, receipt, persist, password)`, `purchaseValidateGoogle(userId, receipt, persist, overrides)`, `purchaseValidateHuawei(...)`, `purchasesList(userId, limit, cursor)`, `purchaseGetByTransactionId(transactionId)` |
| **Subscriptions** | `subscriptionValidateApple(...)`, `subscriptionValidateGoogle(...)`, `subscriptionsList(userId, limit, cursor)`, `subscriptionGetByProductId(userId, productId)` |
| **WebSocket Streams** | `streamUserList(mode, subject, subcontext, label, ...)`, `streamUserJoin(...)`, `streamUserLeave(...)`, `streamSend(mode, subject, subcontext, label, data, presences, reliable)`, `streamSendRaw(...)` |
| **Matchmaker & Match** | `matchCreate(module, params)`, `matchGet(matchId)`, `matchList(limit, authoritative, label, min, max, query)`, `matchSignal(matchId, data)`, `sessionDisconnect(sessionId, reason)` |
| **System Utilities** | `cronNext(expression, timestamp)`, `cronPrev(expression, timestamp)`, `readFile(path)` |

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

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial PRD |
| 1.1 | 2026-07-09 | Engineering | Added Go Initializer interface, Go hook handler signatures with (ctx, logger, db, nk) params |
| 1.2 | 2026-07-09 | Engineering | Expanded API surface detail to specify all missing Social, Group, Tournament, IAP, Subscription, and WebSocket Stream functions |
