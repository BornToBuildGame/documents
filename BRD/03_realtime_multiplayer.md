# BRD-03: Realtime Multiplayer

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Realtime Multiplayer  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a realtime multiplayer infrastructure that supports low-latency game state synchronization, presence tracking, and reliable/unreliable message delivery over persistent connections. This system enables a wide range of game genres including FPS, MOBA, RTS, racing, board games, and card games.
---
## 2. Scope

### In Scope
- Persistent WebSocket connections for realtime communication
- Match creation and lifecycle management
- Reliable and unreliable message delivery
- Match state synchronization
- Player presence within matches
- Spectator support
- Reconnection to active matches
- Client-relay and authoritative match modes

### Out of Scope
- Matchmaking logic (see BRD-02: Multiplayer Matchmaking)
- Server-side game logic execution (see BRD-04: Authoritative Game Server)
- Voice/video communication
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Low-latency communication, flexible match models |
| Players | Smooth gameplay, reliable connections |
| Game Designers | Support for various game genres and modes |
| Network Engineers | Performance, scalability, connection management |
---
## 4. Functional Requirements

### FR-RT-001: WebSocket Connection Management
- **Priority:** Must
- Clients shall establish persistent WebSocket connections to the server.
- Connections shall be authenticated using session tokens.
- The server shall maintain a connection registry for all active clients.
- Heartbeat/ping-pong mechanism to detect stale connections.
- Configurable connection timeouts and keep-alive intervals.

### FR-RT-002: Match Creation
- **Priority:** Must
- Matches shall be creatable through:
  - Matchmaker result (automatic)
  - Direct creation via API (manual)
  - Server-side creation (authoritative)
- Each match shall have a unique match ID.
- Match parameters:
  - `match_id`: UUID
  - `label`: searchable JSON string
  - `max_size`: maximum players
  - `authoritative`: boolean
  - `handler_name`: server runtime handler (if authoritative)

### FR-RT-003: Match Join / Leave
- **Priority:** Must
- Players shall join matches by match ID or match token.
- Players shall be able to leave matches voluntarily.
- Disconnected players shall be removed after a configurable timeout.
- Join conditions can be enforced (e.g., password, invitation, match state).
- Maximum player count shall be enforced.

### FR-RT-004: Reliable Message Delivery
- **Priority:** Must
- Support reliable, ordered message delivery within a match.
- Messages shall be guaranteed to arrive (TCP-based via WebSocket).
- Message format: opcode (int64) + binary payload.
- Max message size: configurable (default: 4 KB, max: 256 KB).

### FR-RT-005: Unreliable Message Delivery
- **Priority:** Should
- Support unreliable (best-effort) message delivery for latency-sensitive data.
- Implementation options: UDP channel or WebSocket with drop semantics.
- Use cases: position updates, input state, non-critical visual effects.

### FR-RT-006: Match State Synchronization
- **Priority:** Must
- The server shall relay state updates between match participants.
- State data format: opcode (int64) + binary data.
- Opcodes allow multiplexing different state types on the same connection.
- In client-relay mode: server forwards messages without inspection.
- In authoritative mode: server processes and validates state (see BRD-04).

### FR-RT-007: Presence Updates
- **Priority:** Must
- The server shall broadcast presence events within a match:
  - Player joined
  - Player left
  - Player disconnected (but reconnect window open)
- Presence data includes: user ID, session ID, username, status, metadata.
- All current match participants shall be retrievable.

### FR-RT-008: Spectator Support
- **Priority:** Should
- Matches shall support spectator slots (separate from player slots).
- Spectators shall receive all match state updates.
- Spectators shall not be able to send match state data.
- Spectators shall be visible in presence (with spectator flag).
- Configurable: allow/disallow spectators per match.

### FR-RT-009: Reconnection Support
- **Priority:** Must
- Disconnected players shall have a grace period to reconnect.
- Configurable reconnect window (default: 30 seconds).
- On reconnect, the player shall:
  - Rejoin the match automatically
  - Receive current match state
  - Receive missed reliable messages (if buffered)
- Reconnection shall restore the player's presence.

### FR-RT-010: Match Listing
- **Priority:** Should
- Clients shall be able to list available matches.
- Filtering by match label (JSON query).
- Pagination support.
- Results include: match ID, label, player count, max size.
- Authoritative matches can control their listing visibility.

### FR-RT-011: Match Lifecycle Events
- **Priority:** Must
- The following events shall be emitted during match lifecycle:
  - `match_created`
  - `match_join`
  - `match_leave`
  - `match_data` (state update)
  - `match_presence` (presence change)
  - `match_terminated`
- Events shall be hookable by server runtime (see BRD-18).

### FR-RT-012: Session-Based Lifetime
- **Priority:** Must
- Realtime multiplayer match participations are strictly session-based. A player's presence in a match is tied to their active WebSocket session.
- If a player's session is terminated (closed socket or connection drop) and the player fails to reconnect within the grace period (see FR-RT-009), the server shall automatically evict the player from the match and notify other participants via match presence leave events.
- In client-relayed matches, if all participants' sessions disconnect, the server shall automatically terminate and garbage collect the match.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | State relay latency ≤ 50ms (server processing, excluding network) |
| **Throughput** | Support 60 state updates per second per match |
| **Scalability** | Support 10,000+ concurrent matches |
| **Connections** | Support 100,000+ concurrent WebSocket connections per node |
| **Message Size** | Default max 4 KB, configurable up to 256 KB |
| **Reconnection** | Reconnect within 30s (configurable) without data loss |
| **Availability** | 99.9% uptime |
| **Monitoring** | Active matches, player counts, message rates, latency metrics |
---
## 6. Match Lifecycle

```
┌─────────────┐
│ Match Created│
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌──────────────┐
│ Players Join │────▶│ Match Active  │
└──────┬──────┘     └──────┬───────┘
       │                   │
       │            ┌──────┴──────┐
       │            │ State Sync   │◀─── tick loop
       │            │ (relay/auth) │
       │            └──────┬──────┘
       │                   │
       ▼                   ▼
┌──────────────┐   ┌──────────────┐
│ Player Leave  │   │ Match End     │
│ / Disconnect  │──▶│ Condition Met │
└──────────────┘   └──────┬───────┘
                          │
                          ▼
                   ┌──────────────┐
                   │ Match Cleanup │
                   │ (persist/log) │
                   └──────────────┘
```
---
## 7. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Session validation for connections |
| BRD-02: Multiplayer Matchmaking | Match creation from matchmaker results |
| BRD-04: Authoritative Game Server | Server-side game logic for authoritative matches |
| BRD-17: Presence System | Player presence tracking |
| BRD-15: Match State Persistence | Saving match state and history |
---
## 8. Acceptance Criteria

- [ ] Clients can establish authenticated WebSocket connections
- [ ] Matches can be created manually or via matchmaker
- [ ] Players can join, leave, and rejoin matches
- [ ] Reliable messages are delivered in order to all match participants
- [ ] State synchronization supports opcode-based multiplexing
- [ ] Presence events (join/leave) are broadcast to all match participants
- [ ] Spectators can observe matches without sending state
- [ ] Disconnected players can reconnect within the grace period
- [ ] Match listing with label-based filtering works
- [ ] Server processes ≤ 50ms relay latency at 60 updates/second
---
## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| WebSocket connection storms | Server overload | Medium | Connection rate limiting; gradual reconnect |
| Message flooding by malicious clients | Performance degradation | Medium | Per-client message rate limits; opcode validation |
| State desync (client-relay mode) | Inconsistent gameplay | High | Periodic state reconciliation; checksums |
| High-latency regions | Poor gameplay experience | Medium | Regional server deployment; latency reporting |
| Large match sizes | Memory/CPU pressure | Low | Max match size limits; horizontal scaling |
---
## 10. Future Considerations

- UDP transport for unreliable channels (QUIC / WebTransport)
- Server-side interest management (only send relevant state)
- Delta compression for state updates
- Match recording and replay system
- Load balancing across match server nodes
- Cross-server match migration (live migration)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-03](../PRD/03_realtime_multiplayer.md) (API Surface & Interface Specification)
- [TDD-03](../TDD/03_realtime_multiplayer.md) (Database Schema & Technical Implementation Design)
