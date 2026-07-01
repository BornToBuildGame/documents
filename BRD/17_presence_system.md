# BRD-17: Presence System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Presence System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a real-time presence system that tracks the online status, current activity, match participation, party status, and other live state information of connected users. Presence is foundational to social features, matchmaking, and real-time coordination.
---
## 2. Scope

### In Scope
- Online/offline status tracking
- Activity tracking (in match, in menu, in queue, etc.)
- Status messages and custom presence data
- Presence subscriptions (follow/unfollow)
- Presence events (join, leave, update)
- Match presence
- Party presence
- Chat channel presence
- Global presence tracking

### Out of Scope
- Offline last-seen tracking (simple timestamp, not real-time)
- Location-based presence (GPS)
- Push notifications for offline users (see BRD-11)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Players | See friends' status, know who's online |
| Game Developers | Track active users, coordinate real-time features |
| Game Designers | Design presence-aware features |
| Server Engineers | Presence scalability and distribution |
---
## 4. Functional Requirements

### FR-PR-001: Online Status Tracking
- **Priority:** Must
- The system shall track whether a user is online or offline.
- A user is online when they have at least one active connection (WebSocket).
- A user goes offline when all connections are closed (after grace period).
- Grace period for disconnects: configurable (default: 15 seconds).

### FR-PR-002: Presence Data
- **Priority:** Must
- Each online user's presence includes:

| Field | Description |
|-------|-------------|
| `user_id` | User identifier |
| `session_id` | Active session identifier |
| `username` | Display username |
| `node` | Server node hosting the connection |
| `status` | Custom status JSON (set by user/game) |

### FR-PR-003: Status Updates
- **Priority:** Must
- Users shall update their own status via WebSocket.
- Status is a JSON object (developer-defined schema).
- Examples:
  ```json
  { "state": "in_menu" }
  { "state": "in_match", "match_id": "abc123", "mode": "ranked" }
  { "state": "away", "message": "AFK" }
  ```
- Status updates are broadcast to all subscribers.

### FR-PR-004: Presence Subscriptions
- **Priority:** Must
- Users shall subscribe (follow) to other users' presence.
- On subscription:
  - Receive the target's current presence (if online)
  - Receive future presence events (status change, disconnect)
- Users shall unsubscribe (unfollow) to stop receiving events.
- Batch follow/unfollow supported.
- Automatic follow: friends are auto-followed on login (configurable).

### FR-PR-005: Presence Events
- **Priority:** Must
- Events broadcast to subscribers:
  - `join`: user came online
  - `leave`: user went offline
  - `update`: user's status changed
- Event payload includes full presence data.
- Delivered via WebSocket in real-time.

### FR-PR-006: Match Presence
- **Priority:** Must
- When a user joins a match, their presence within the match is tracked.
- Match presence events:
  - Player joined match
  - Player left match
  - Player's match state changed
- All match participants receive presence events.
- Match presence is separate from global presence.

### FR-PR-007: Party Presence
- **Priority:** Must
- Party members' presence is tracked within the party.
- Party presence events: member joined, member left, member status changed.
- All party members receive presence events.

### FR-PR-008: Chat Channel Presence
- **Priority:** Should
- Track which users are currently active in a chat channel.
- Channel presence events: user joined channel, user left channel.
- Used for: typing indicators, active user counts.

### FR-PR-009: Presence Query
- **Priority:** Should
- Query the presence of specific users (batch lookup).
- Input: list of user IDs.
- Output: presence data for each online user (absent users are offline).

### FR-PR-010: Global Presence Metrics
- **Priority:** Should
- Track aggregate presence metrics:
  - Total online users
  - Users per region/node
  - Users per activity state
- Available via admin API for monitoring.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | Presence events delivered within 200ms |
| **Scalability** | Support 1,000,000+ concurrent presences |
| **Fan-out** | Support 1,000+ subscribers per user's presence |
| **Memory** | Efficient in-memory presence tracking |
| **Availability** | 99.9% uptime |
| **Consistency** | Eventual consistency for cross-node presence (≤ 1s) |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | User identity and session |
| BRD-03: Realtime Multiplayer | Match presence |
| BRD-07: Friends System | Auto-follow friends |
| BRD-08: Parties | Party presence |
| BRD-10: Chat System | Channel presence |
| BRD-19: API Layer | WebSocket transport |
---
## 7. Acceptance Criteria

- [ ] Users are tracked as online when WebSocket is connected
- [ ] Users are tracked as offline after all connections close (with grace period)
- [ ] Status updates are broadcast to all subscribers in real-time
- [ ] Presence subscriptions (follow/unfollow) work correctly
- [ ] Match, party, and chat presence are tracked separately
- [ ] Presence events include user_id, session_id, username, and status
- [ ] Batch presence queries return correct results for online users
- [ ] Presence delivery latency ≤ 200ms
- [ ] System supports 1M+ concurrent presences
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Presence fan-out storm (celebrity user) | Server overload | Medium | Fan-out limits; batched delivery |
| Cross-node presence consistency | Stale presence data | Medium | Gossip protocol; event-driven sync |
| Memory exhaustion (large concurrent) | Node crash | Low | Presence eviction; per-node limits |
| Flapping (rapid connect/disconnect) | Event spam | Medium | Grace period; debounce presence events |
---
## 9. Future Considerations

- Rich presence (game-specific detailed status)
- Invisible / Do Not Disturb mode
- Presence history (when was user last online)
- Presence-based notifications (notify when friend comes online)
- Geographic presence (server region)
- Presence analytics (peak online times, activity patterns)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-17](../PRD/17_presence_system.md) (API Surface & Interface Specification)
- [TDD-17](../TDD/17_presence_system.md) (Database Schema & Technical Implementation Design)
