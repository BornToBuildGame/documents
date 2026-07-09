# Ultimate Game Engine Server Implementation Plan & Status Checklist

This document details the implementation plan and tracks the progress of the **Ultimate Game Engine Multiplayer Game Server** backend. 

Currently, the code repository at `ultimate-game-server` is in the bootstrap phase (contains only `LICENSE` and `.git`). Therefore, all implementation and testing tasks are marked as **Pending**.

---

## Status Summary

| Phase | Component / Feature Area | Target Documents | Status |
|---|---|---|---|
| 1 | Project Bootstrap & Database Infrastructure | [TDD-20](./TDD/20_database_infrastructure.md) | 🟩 Completed |
| 2 | Identity & Session Layer | [TDD-01](./TDD/01_user_authentication.md) | 🟩 Completed |
| 3 | Transport & API Gateway Layer | [TDD-19](./TDD/19_api_layer.md) | 🟩 Completed |
| 4 | Real-time Socket Gateway & Presence | [TDD-03](./TDD/03_realtime_multiplayer.md), [TDD-17](./TDD/17_presence_system.md) | 🟩 Completed |
| 5 | Extensibility Runtime & VM Sandbox | [TDD-14](./TDD/14_rpc_custom_apis.md), [TDD-18](./TDD/18_server_runtime_hooks.md) | 🟩 Completed |
| 6 | Matchmaking & Authoritative Loop | [TDD-02](./TDD/02_multiplayer_matchmaking.md), [TDD-04](./TDD/04_authoritative_game_server.md), [TDD-15](./TDD/15_match_state_persistence.md) | 🟩 Completed |
| 7 | Social Graph & Groups | [TDD-07](./TDD/07_friends_system.md), [TDD-08](./TDD/08_parties.md), [TDD-09](./TDD/09_guilds_clans.md), [TDD-16](./TDD/16_groups.md) | 🟩 Completed |
| 8 | Messaging, Chat & Notifications | [TDD-10](./TDD/10_chat_system.md), [TDD-11](./TDD/11_notifications.md) | 🟩 Completed |
| 9 | Storage Engine & Economy Ledger | [TDD-12](./TDD/12_storage_engine.md), [TDD-13](./TDD/13_economy_system.md), [TDD-21](./TDD/21_iap_validation.md) | 🟩 Completed |
| 10 | Console Admin & Monitoring | [TDD-22](./TDD/22_console_admin.md) | 🟩 Completed |
| 11 | Leaderboard & Tournament Engine | [TDD-05](./TDD/05_leaderboards.md), [TDD-06](./TDD/06_tournaments.md) | 🟩 Completed |
| 12 | Ephemeral Party Gateway | [TDD-08](./TDD/08_parties.md) | 🟩 Completed |
| 13 | Distributed Multi-Node Scaling & Interceptors | [TDD-03](./TDD/03_realtime_multiplayer.md), [TDD-04](./TDD/04_authoritative_game_server.md), [TDD-18](./TDD/18_server_runtime_hooks.md) | 🟩 Completed |

---

## User Review Required

> [!IMPORTANT]
> The database schema design (`database-design.md`) dictates the usage of PostgreSQL as the transactional database and Redis for transient memory stores. All tables, triggers, and indices listed in the design must be migrated and validated during Phase 1. 

---

## Implementation Checklist with Testing & Verification

### Phase 1: Project Bootstrap & Database Infrastructure (TDD-20)
* **Implementation Tasks:**
  - `[x]` Initialize Go project module structure in `ultimate-game-server`
  - `[x]` Set up local Docker Compose configuration (`postgres`, `redis`)
  - `[x]` Implement PGX connection pool manager with exponential startup backoff logic
  - `[x]` Build transactional migration runner checking the `schema_version` table
  - `[x]` Write initial database DDL script (`0001_initial_schema.sql`) creating:
    - Tables: `schema_version`
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Verify exponential backoff timing logic (progressive delays scale correctly).
  - `[x]` **Integration Test:** Spin up PostgreSQL container via `testcontainers-go` and verify connection retry.
  - `[x]` **Integration Test:** Execute migrations against PostgreSQL container and verify `schema_version` record updates transactionally.
  - `[x]` **Verify command:** `go test -v ./internal/database/...`

### Phase 2: Identity & Session Layer (TDD-01)
* **Implementation Tasks:**
  - `[x]` Add DDL schema for user profiles: `users`, `user_device`, `user_tombstone`
  - `[x]` Add indices: `idx_users_email`, `idx_users_apple_id`, `idx_users_google_id`, `idx_users_facebook_id`, `idx_users_gamecenter_id`, `idx_users_steam_id`, `idx_users_custom_id`, `idx_users_facebook_instant_game_id`, `idx_users_display_name_trgm`, `idx_user_device_user_id`
  - `[x]` Implement authentication handlers: email password verification (`bcrypt`) and custom device token sign-in
  - `[x]` Implement OAuth token verification and account linking (Apple, Google, Steam, Facebook, GameCenter)
  - `[x]` Implement stateless HS256/RS256 JWT access token generator
  - `[x]` Implement refresh token rotation and token reuse detection (tombstoning)
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Verify JWT token creation, signature verification, and claims parsing. Test token expiry logic.
  - `[x]` **Unit Test:** Verify password hash verification returns correct errors for invalid input.
  - `[x]` **Integration Test:** Connect to Testcontainers PostgreSQL to verify user registration, duplicate username/email constraints, device token linking, and soft delete tombstone generation.
  - `[x]` **Integration Test:** Verify refresh token rotation invalidates the old token and returns a conflict error on repeated reuse of the same token.
  - `[x]` **Verify command:** `go test -v ./internal/auth/...`

### Phase 3: Transport & API Gateway Layer (TDD-19)
* **Implementation Tasks:**
  - `[x]` Build HTTP REST router and gRPC server binds on ports `7350` and `7349`
  - `[x]` Mount HTTP/gRPC security middlewares (CORS, secure headers, 4KB payload read limit checks)
  - `[x]` Implement Token Bucket rate limiter (local memory for single-node, Redis-backed for multi-node)
  - `[x]` Implement caller context JWT interceptor middleware
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Verify Token Bucket algorithm refill rates, max limit thresholds, and consumption returns.
  - `[x]` **Integration Test (Redis):** Spin up Redis container via `testcontainers-go` and verify distributed rate limit checks across nodes.
  - `[x]` **Integration Test:** Run local server mock and verify client HTTP requests exceeding 4KB are rejected with a payload size limit error.
  - `[x]` **Integration Test:** Verify JWT token headers are read, context is parsed, and requests lacking valid tokens receive HTTP `401 Unauthorized`.
  - `[x]` **Verify command:** `go test -v ./internal/api/...`

### Phase 4: Real-time Socket Gateway & Presence (TDD-03, TDD-17)
* **Implementation Tasks:**
  - `[x]` Create HTTP WebSocket connection upgrade handler (`GET /v2/stream`)
  - `[x]` Build thread-safe connection registry manager
  - `[x]` Implement ping/pong heartbeat validation (15s ping, 30s pong deadline) with a 30s session recovery grace window
  - `[x]` Build presence tracker (using local registry for single-node, Redis hashes for multi-node)
  - `[x]` Integrate `roaring` bitmap checks for social presence lookups
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Test connection registry concurrency: thread-safe mapping additions, fetches, and deletions.
  - `[x]` **Integration Test (WS + Redis):** Simulate WebSocket clients, trigger pings, verify timeout disconnects when client misses the pong deadline.
  - `[x]` **Integration Test:** Verify session recovery: client disconnects, reconnects with same session token within 30s grace window, and is reinstated without presence loss.
  - `[x]` **Integration Test:** Test presence event dispatching: verify that when User A goes online, User B (friend) receives WebSocket status updates.
  - `[x]` **Verify command:** `go test -v ./internal/socket/... ./internal/presence/...`

### Phase 5: Extensibility Runtime & VM Sandbox (TDD-14, TDD-18)
* **Implementation Tasks:**
  - `[x]` Integrate VM isolation layer (Lua runtime or JavaScript QuickJS sandbox)
  - `[x]` Enforce execution limit quotas (64MB memory heap limit, 10M instructions CPU limit)
  - `[x]` Implement runtime hooks registrations (before/after request handlers and server RPCs)
  - `[x]` Define `Logger` and `RuntimeModule` interfaces for Go native runtime
  - `[x]` Define `Initializer` interface with full registration method set (RegisterRpc, RegisterBeforeRt, RegisterAfterRt, RegisterMatch, RegisterEvent, etc.)
  - `[x]` Update `RPCHandler`, `BeforeHook`, `AfterHook` type signatures with `(ctx, logger, db, nk)` parameters
  - `[x]` Implement `GoRuntimeManager` with `.so` plugin loading via `plugin.Open()`
  - `[x]` Implement `InitModule` symbol resolution and type-assertion for loaded plugins
  - `[x]` Implement panic recovery wrapper (`recover()`) for all Go runtime handler invocations
  - `[x]` Implement `goInitializer` that captures registrations into `HookRegistry` during `InitModule`
  - `[x]` Implement runtime precedence resolution (Go → Lua → JavaScript)
  - `[x]` Implement SHA-256 checksum verification for `.so` plugin files
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Load scripts with infinite loops or heavy memory allocations and verify execution halts safely at instructions budget or memory limit boundaries.
  - `[x]` **Unit Test:** Verify custom RPC scripts execute in sandbox isolates, returning correct output values without thread-safety race conditions.
  - `[x]` **Unit Test:** Verify Go RPC handlers with full `(ctx, logger, db, nk, payload)` signature execute correctly.
  - `[x]` **Unit Test:** Verify `goInitializer` registers RPCs, before/after hooks, and events into `HookRegistry`.
  - `[x]` **Unit Test:** Verify panic recovery: a panicking Go RPC handler returns an error instead of crashing the process.
  - `[x]` **Unit Test:** Verify `GoRuntimeManager.HasRPC/HasBeforeHook/HasAfterHook` return correct results.
  - `[x]` **Verify command:** `go test -v ./internal/runtime/...`

### Phase 6: Matchmaking & Authoritative Loop (TDD-02, TDD-04, TDD-15)
* **Implementation Tasks:**
  - `[x]` Implement matchmaking queue submit, ticket validation, and cancellation handlers
  - `[x]` Build asynchronous matchmaking tick loop (using Redlock for multi-node coordinator locking)
  - `[x]` Implement MMR skill-window expansion curves based on ticket queue wait time
  - `[x]` Implement Authoritative Match loops running on isolated goroutine workers (30Hz tick rate updates)
  - `[x]` Build peer-to-peer inter-node gRPC routing mesh to forward WS packets across nodes
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Verify MMR expansion curve progression matching wait time bounds.
  - `[x]` **Integration Test (Redis):** Submit matchmaking tickets and verify match loop matching evaluates correctly. Test ticket removal post-matching.
  - `[x]` **Integration Test (Redlock):** Spin up two mock nodes and verify only one node gains the coordinator lock per matchmaking tick.
  - `[x]` **Integration Test (Match Loop):** Spin up authoritative match goroutines, input simulated commands, and assert match tick logic executes at 30Hz, persisting outcomes to DB.
  - `[x]` **Verify command:** `go test -v ./internal/matchmaker/... ./internal/match/...`

### Phase 7: Social Graph & Groups (TDD-07, TDD-08, TDD-09, TDD-16)
* **Implementation Tasks:**
  - `[x]` Add DDL schema for: `user_edge`, `groups`, `group_edge`
  - `[x]` Add indices: `idx_user_edge_fk_destination_id`, `idx_groups_edge_count_update_time_id`, `idx_groups_update_time_edge_count_id`, `idx_groups_name_active` (conditional index on active groups)
  - `[x]` Implement directed social edge transaction dispatches (mutual friend links, invites, blocklists)
  - `[x]` Implement group creation, listing, membership additions, and role demographic validation
* **Testing & Verification Tasks:**
  - `[x]` **Integration Test:** Verify friend invitation transactional states: inserting edge `A -> B` and `B -> A` inside a transaction. Test acceptance of friend request.
  - `[x]` **Integration Test:** Verify user block logic removes the reciprocal relation edge to avoid interaction.
  - `[x]` **Integration Test:** Test group membership boundaries: assert group creation constraint checks open/close, edge counts updates, and maximum member limitations.
  - `[x]` **Integration Test:** Test role-based privilege checks: verify Demote, Promote, and Kick checks prevent operations when executed by normal members.
  - `[x]` **Verify command:** `go test -v ./internal/social/...`

### Phase 8: Messaging, Chat & Notifications (TDD-10, TDD-11)
* **Implementation Tasks:**
  - `[x]` Add DDL schema for: `message`, `notification`
  - `[x]` Add indices: `idx_message_sender`, `idx_notification_user_id`
  - `[x]` Implement chat routing handlers (direct, group, room streams)
  - `[x]` Implement notification triggers (real-time WS pushes and external APNS/FCM dispatches)
* **Testing & Verification Tasks:**
  - `[x]` **Integration Test:** Send direct and group chat messages. Validate payload persistence in `message` database table.
  - `[x]` **Integration Test:** Emit user notification: assert notification record is inserted into `notification` table and immediate WebSocket payload dispatch occurs for online target users.
  - `[x]` **Verify command:** `go test -v ./internal/chat/... ./internal/notification/...`

### Phase 9: Storage Engine & Economy Ledger (TDD-12, TDD-13, TDD-21)
* **Implementation Tasks:**
  - `[x]` Add DDL schema for: `storage`, `wallet_ledger`, `purchase`, `subscription`
  - `[x]` Add indices: `idx_storage_collection_read_user_id_key`, `idx_storage_collection_read_key_user_id`, `idx_storage_collection_user_id_read_key`, `idx_storage_auto_index_fk_user_id`, `idx_storage_value_gin` (GIN index on jsonb values), `idx_wallet_ledger_user_history`, `idx_purchase_time_user_id_transaction_id`, `idx_subscription_time_user_id_transaction_id`
  - `[x]` Implement storage write/read handlers with MD5-based Optimistic Concurrency Control (OCC)
  - `[x]` Implement atomic wallet balance mutations (`SELECT FOR UPDATE` on `users.wallet` column)
  - `[x]` Implement receipt validations and purchase ledger logging
* **Testing & Verification Tasks:**
  - `[x]` **Integration Test:** Read/Write objects to the storage engine. Verify OCC write rejects updates with an old version hash, returning a `409 Conflict` error.
  - `[x]` **Integration Test (Concurrency):** Run concurrent ledger wallet balance mutations. Assert database pessimistic lock `FOR UPDATE` prevents double-spends and enforces non-negative wallet constraints under parallel requests. Verify ledger entries are correctly created.
  - `[x]` **Verify command:** `go test -v ./internal/storage/... ./internal/economy/...`

### Phase 10: Console Admin & Monitoring (TDD-22)
* **Implementation Tasks:**
  - `[x]` Add DDL schema for: `console_user`, `console_audit_log`, `console_acl_template`, `setting`, `users_notes`
  - `[x]` Add indices: `idx_users_notes_user_id`
  - `[x]` Set up isolated Admin REST router binding on port `7351`
  - `[x]` Implement Admin JWT authentication using secure `HttpOnly` cookies and MFA authentication (TOTP)
  - `[x]` Integrate `bleve` full-text search engine index for log queries and player profile lookup
  - `[x]` Implement administrative mutation logs mapping to `console_audit_log`
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Verify MFA TOTP validation checks with valid/invalid secret keys.
  - `[x]` **Integration Test:** Authenticate console admin user. Verify HTTP endpoint cookie check enforces isolated secure cookies.
  - `[x]` **Integration Test:** Perform admin-only moderation updates (such as player ban) and assert log entries populate the `console_audit_log` table.
  - `[x]` **Integration Test:** Perform full-text player lookups via administrative API, verifying indices return expected query results.
  - `[x]` **Verify command:** `go test -v ./internal/console/...`

### Phase 11: Leaderboard & Tournament Engine (TDD-05, TDD-06)
* **Implementation Tasks:**
  - `[x]` Create `0007_leaderboard_tournament_schema.sql` migration defining `leaderboard` and `leaderboard_record` tables, constraints, and index structures
  - `[x]` Implement `internal/leaderboard` core package with score operator logic (best, set, increment, decrement)
  - `[x]` Implement local thread-safe O(log N) in-memory ranking cache for fast placements lookup
  - `[x]` Implement Redis Pub/Sub invalidation listener to clear/rebuild local rank caches lazily
  - `[x]` Implement `internal/tournament` cron scheduler checking active occurrences every 30s
  - `[x]` Implement `OnTournamentEnd` hook invocation triggering reward wallet mutations inside PostgreSQL database transactions
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Submit scores with different operators and verify calculated leaderboard placements.
  - `[x]` **Integration Test:** Verify cron parser triggers active tournament states and runs tournament end reward hooks.
  - `[x]` **Verify command:** `go test -v ./internal/leaderboard/... ./internal/tournament/...`

### Phase 12: Ephemeral Party Gateway (TDD-08)
* **Implementation Tasks:**
  - `[x]` Implement `internal/party` core package with thread-safe `PartySession` and `PartyMember` state structures
  - `[x]` Implement socket command message handlers (`party_create`, `party_invite`, `party_join`, `party_leave`) in `internal/socket`
  - `[x]` Implement leader promotion logic triggering when current leader disconnects or leaves the party
  - `[x]` Implement invitation lifecycle maps and auto-expiration ticks
* **Testing & Verification Tasks:**
  - `[x]` **Unit Test:** Verify party creation, invitation limits, and leader promotion sorting rules.
  - `[x]` **Integration Test:** Simulate client connections joining, leaving, and sending messages over party WebSocket streams.
  - `[x]` **Verify command:** `go test -v ./internal/party/...`

### Phase 13: Distributed Multi-Node Scaling & Interceptors (TDD-03, TDD-04, TDD-18)
* **Implementation Tasks:**
  - `[x]` Implement distributed Redis-backed socket message relaying for multi-node connection architectures
  - `[x]` Implement authoritative match labeling index registry and active search query endpoints
  - `[x]` Implement gRPC peer-to-peer input forwarding mesh routing client commands between server nodes
  - `[x]` Wire runtime before/after hooks registry execution interceptors into all main database and authentication API endpoints
* **Testing & Verification Tasks:**
  - `[x]` **Integration Test:** Run multi-node cluster tests, forwarding inputs over gRPC and verifying state is relayed across nodes.
  - `[x]` **Integration Test:** Verify custom before/after runtime hooks execute and intercept auth/storage endpoints correctly.
  - `[x]` **Verify command:** `go test -v ./internal/socket/... ./internal/match/...`

---

## Global Verification Commands

Use the following commands in the workspace root to check for race conditions and total test coverage:

```bash
# Run all unit and integration tests with race detector
go test -race -tags=integration -v ./...

# Run test coverage checks for all packages
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
```
*Note: A coverage target of ≥ 80% line coverage is required per package before marking tasks complete.*


