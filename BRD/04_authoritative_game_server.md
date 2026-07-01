# BRD-04: Authoritative Game Server

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Authoritative Game Server  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for an authoritative game server where game logic executes server-side, preventing cheating and ensuring fair gameplay. The server validates all player actions, owns game state, and distributes results to clients.
---
## 2. Scope

### In Scope
- Server-side game logic execution
- Match handler registration and lifecycle
- Input validation and action authorization
- Server-authoritative state management
- Tick-based game loop on the server
- Anti-cheat through server ownership of truth
- Multiple runtime language support (Lua, TypeScript, Go)

### Out of Scope
- Client-relay multiplayer (see BRD-03 for relay mode)
- Client-side prediction and reconciliation (client SDK concern)
- Dedicated game server orchestration (future enterprise feature)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Write game logic in preferred language; debug server-side code |
| Players | Fair gameplay, cheat-free environment |
| Anti-Cheat Team | Server-authoritative validation |
| Game Designers | Deterministic game rules, balanced mechanics |
---
## 4. Functional Requirements

### FR-AG-001: Match Handler Registration
- **Priority:** Must
- Developers shall register named match handlers on the server runtime.
- Each handler defines the logic for a specific game type.
- Handler interface:
  - `match_init(context, params)` → initial state
  - `match_join_attempt(context, state, presence)` → allow/deny
  - `match_join(context, state, presences)` → updated state
  - `match_leave(context, state, presences)` → updated state
  - `match_loop(context, state, tick, messages)` → updated state + broadcasts
  - `match_signal(context, state, data)` → updated state + response
  - `match_terminate(context, state, grace_period)` → cleanup

### FR-AG-002: Server-Side Game Loop
- **Priority:** Must
- The server shall execute a tick-based game loop for each authoritative match.
- Configurable tick rate per match (default: 10 ticks/second, max: 60).
- Each tick:
  1. Collect all incoming player messages since last tick
  2. Call `match_loop` with accumulated messages
  3. Broadcast state updates returned by the handler
  4. Manage match lifecycle (end conditions, timeouts)
- The game loop shall execute within a single goroutine/thread per match for consistency.

### FR-AG-003: Input Validation
- **Priority:** Must
- All player inputs shall be received and validated server-side.
- The server shall reject invalid inputs (out-of-turn actions, impossible moves).
- Input messages use opcode + payload format.
- The handler decides what constitutes a valid input.

### FR-AG-004: Server-Authoritative State
- **Priority:** Must
- The server shall own and manage the canonical game state.
- Clients receive state updates but cannot directly modify server state.
- State shall be an opaque object managed entirely by the match handler.
- State size limit: configurable (default: 1 MB per match).

### FR-AG-005: Anti-Cheat by Design
- **Priority:** Must
- Damage calculation shall be server-side.
- Item ownership and transactions shall be server-validated.
- Movement and position shall be server-validated (where applicable).
- Random number generation shall be server-side.
- The server shall detect and flag suspicious patterns:
  - Impossible action speed
  - Out-of-bounds values
  - Sequence anomalies

### FR-AG-006: Runtime Language Support
- **Priority:** Must
- Match handlers shall be writable in:
  - **Lua** — embedded scripting, hot-reloadable
  - **TypeScript** — compiled to JavaScript, runs on embedded runtime
  - **Go** — compiled as plugins or built into the server binary
- All languages shall have access to the same server APIs:
  - Storage read/write
  - User account access
  - Leaderboard operations
  - Notification dispatch
  - Matchmaker operations
  - Wallet updates

### FR-AG-007: Match Signals
- **Priority:** Should
- External systems shall be able to send signals to active matches.
- Signals are arbitrary string payloads delivered to the `match_signal` handler.
- Use cases:
  - Admin commands (force end match, announce message)
  - Tournament system integration
  - Event triggers
- The handler returns a response to the signal sender.

### FR-AG-008: Deferred Messages
- **Priority:** Should
- The match handler shall support scheduling deferred messages.
- A deferred message is delivered to `match_loop` after a specified delay.
- Use cases: timed events, cooldowns, phase transitions.
- Deferred messages shall survive individual ticks but not server restart (in-memory).

### FR-AG-009: Match Termination
- **Priority:** Must
- Matches shall terminate when:
  - The handler returns a terminate signal
  - All players leave (configurable: terminate immediately or after timeout)
  - An admin signal forces termination
  - The server shuts down (graceful termination with `match_terminate` callback)
- A grace period shall be provided for cleanup on server shutdown.

### FR-AG-010: Match Labeling
- **Priority:** Should
- Authoritative matches shall be able to update their label (JSON string) dynamically.
- Labels are used for match listing and discovery (see BRD-03).
- Labels can represent: game mode, current phase, player count, join eligibility.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Tick Performance** | Each tick shall complete within 1000/tick_rate ms (e.g., 100ms at 10 tps) |
| **Scalability** | Support 5,000+ concurrent authoritative matches per node |
| **Determinism** | Same inputs + same state → same outputs (within a single server) |
| **Isolation** | Match failures shall not affect other matches |
| **Hot Reload** | Lua/TypeScript handlers should support hot reload without restart |
| **Logging** | Per-match logging with configurable verbosity |
| **Monitoring** | Tick duration, match count, handler errors, message rates |
---
## 6. Match Handler Interface

### Lua Example
```lua
local M = {}

function M.match_init(context, params)
  local state = {
    players = {},
    phase = "waiting",
    round = 0
  }
  local tick_rate = 10
  local label = json.encode({ mode = params.mode or "default" })
  return state, tick_rate, label
end

function M.match_join_attempt(context, dispatcher, tick, state, presence, metadata)
  if #state.players >= 4 then
    return state, false, "Match is full"
  end
  return state, true
end

function M.match_join(context, dispatcher, tick, state, presences)
  for _, p in ipairs(presences) do
    state.players[p.user_id] = { score = 0, ready = false }
  end
  return state
end

function M.match_leave(context, dispatcher, tick, state, presences)
  for _, p in ipairs(presences) do
    state.players[p.user_id] = nil
  end
  return state
end

function M.match_loop(context, dispatcher, tick, state, messages)
  for _, msg in ipairs(messages) do
    -- Validate and process player input
    -- Update game state
  end
  -- Broadcast state to all players
  dispatcher.broadcast_message(1, json.encode(state))
  return state
end

function M.match_terminate(context, dispatcher, tick, state, grace_seconds)
  -- Save final state, distribute rewards
  return nil
end

return M
```
---
## 7. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-03: Realtime Multiplayer | WebSocket transport, match infrastructure |
| BRD-01: User Authentication | Player identity for match actions |
| BRD-12: Storage Engine | Persistent state access from handlers |
| BRD-05: Leaderboards | Score submission from handlers |
| BRD-18: Server Runtime & Hooks | Runtime execution environment |
| BRD-13: Economy System | Wallet and item operations from handlers |
---
## 8. Acceptance Criteria

- [ ] Match handlers can be registered in Lua, TypeScript, and Go
- [ ] Server executes a tick-based game loop at the configured rate
- [ ] Player inputs are validated server-side before state updates
- [ ] The server is the authoritative source of game state
- [ ] Match handlers have access to storage, leaderboards, wallets, and notifications
- [ ] Matches terminate properly when end conditions are met
- [ ] Signals can be sent to active matches from external systems
- [ ] Match labels can be dynamically updated
- [ ] Individual match failures are isolated and don't crash the server
- [ ] 5,000+ concurrent authoritative matches run without degradation
---
## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Buggy handler code crashing the server | All matches on node affected | Medium | Sandbox execution; per-match error isolation |
| Handler infinite loops | Thread/goroutine starvation | Medium | Tick timeout enforcement; resource limits |
| High tick rate + many matches | CPU exhaustion | Medium | Tick rate caps; match-per-node limits |
| State size growth | Memory pressure | Low | State size limits; periodic cleanup |
| Language runtime overhead (Lua/JS) | Higher latency vs. native | Low | Go plugins for performance-critical games |
---
## 10. Future Considerations

- Deterministic lockstep mode for competitive games
- Match simulation/replay (re-run inputs against handler)
- Distributed match state (multi-node matches for MMOs)
- GPU-accelerated physics on server
- Automated handler testing framework
- Match handler versioning (run old and new simultaneously)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-04](../PRD/04_authoritative_game_server.md) (API Surface & Interface Specification)
- [TDD-04](../TDD/04_authoritative_game_server.md) (Database Schema & Technical Implementation Design)
