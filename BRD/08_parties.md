# BRD-08: Parties

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Parties  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a party system enabling players to form temporary groups for co-op play, party matchmaking, party chat, and coordinated game activities. Parties are session-based groups that exist while members are online.
---
## 2. Scope

### In Scope
- Party creation and management
- Party invitations and join requests
- Party chat
- Party matchmaking integration
- Party leadership and promotion
- Party presence tracking
- Party size limits

### Out of Scope
- Persistent groups (see BRD-09: Guilds, BRD-16: Groups)
- Voice chat within parties
- Cross-server parties (future enterprise feature)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Players | Group up with friends, queue together |
| Game Designers | Co-op gameplay, social engagement |
| Game Developers | Party-aware matchmaking, easy API |
| UX Designers | Party invite flow, party HUD |
---
## 4. Functional Requirements

### FR-PT-001: Party Creation
- **Priority:** Must
- Any online player shall create a party.
- The creator becomes the party leader.
- Party parameters:
  - `party_id`: auto-generated UUID
  - `open`: boolean (true = anyone can join; false = invite-only)
  - `max_size`: configurable (default: 4, max: 16)
- A player can only be in one party at a time.

### FR-PT-002: Party Invitation
- **Priority:** Must
- The party leader (or any member, if configured) can invite players.
- Invitations are delivered via WebSocket (real-time).
- Invitations expire after a configurable timeout (default: 60 seconds).
- Invited players can accept or decline.
- Blocked users cannot be invited.

### FR-PT-003: Party Join
- **Priority:** Must
- For open parties: any player can join by party ID.
- For closed parties: only invited players can join.
- Join is rejected if party is full.
- On join: all party members receive a presence event.

### FR-PT-004: Party Leave
- **Priority:** Must
- Any member can leave the party voluntarily.
- Disconnected members are removed after a grace period (configurable, default: 30s).
- If the leader leaves:
  - Leadership is automatically promoted to the longest-active member.
  - If no members remain, the party is dissolved.

### FR-PT-005: Party Leader Actions
- **Priority:** Must
- The party leader can:
  - Invite players
  - Kick members
  - Promote another member to leader
  - Close/open the party
  - Start party matchmaking
- Leadership can be transferred explicitly.

### FR-PT-006: Party Chat
- **Priority:** Must
- All party members shall have access to a party chat channel.
- Party chat is real-time via WebSocket.
- Chat messages include: sender, content, timestamp.
- Chat history is not persisted after party dissolution (ephemeral).
- Integration with Chat System (BRD-10) for consistency.

### FR-PT-007: Party Matchmaking
- **Priority:** Must
- The party leader can submit the party as a unit into matchmaking.
- All party members are included in a single matchmaking ticket.
- Party skill rating: configurable (average, max, weighted).
- When a match is found, all party members are placed together.
- If any party member declines, the ticket is cancelled.
- Integration with Matchmaking (BRD-02).

### FR-PT-008: Party Presence
- **Priority:** Must
- All party members can see who is in the party.
- Presence updates (join, leave, promote) are broadcast to all members.
- Presence data: user ID, username, display name, status, is_leader.
- Integration with Presence System (BRD-17).

### FR-PT-009: Party Data
- **Priority:** Should
- Arbitrary key-value data can be stored on the party (in-memory).
- Use cases: selected game mode, ready status, character selection.
- Any member can read; leader (or all, if configured) can write.
- Data is lost when the party dissolves.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | Party operations (create, join, leave) ≤ 50ms |
| **Scalability** | Support 100,000+ concurrent parties |
| **Ephemeral** | Parties exist only while members are online |
| **Real-time** | All party events delivered within 100ms via WebSocket |
| **Availability** | 99.9% uptime |
---
## 6. Party Lifecycle

```
┌──────────────┐
│ Party Created │
│ (leader only) │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│ Invite/Open  │────▶│ Members Join  │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│ Party Chat   │     │ Ready Up /   │
│ Coordination │     │ Select Mode  │
└──────┬───────┘     └──────┬───────┘
       │                    │
       └────────┬───────────┘
                │
                ▼
       ┌──────────────┐
       │ Matchmaking   │
       │ (as party)    │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │ Match Found   │
       │ All placed    │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │ In Match     │
       │ (party stays) │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │ Dissolve or   │
       │ Continue      │
       └──────────────┘
```
---
## 7. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Player identity |
| BRD-02: Multiplayer Matchmaking | Party matchmaking |
| BRD-03: Realtime Multiplayer | Match placement |
| BRD-07: Friends System | Invite friends to party |
| BRD-10: Chat System | Party chat channel |
| BRD-17: Presence System | Party presence tracking |
---
## 8. Acceptance Criteria

- [ ] Players can create a party and become leader
- [ ] Players can invite others; invitations are delivered in real-time
- [ ] Open parties allow anyone to join; closed parties require invitation
- [ ] Party size limits are enforced
- [ ] Leadership transfers on leader departure
- [ ] Party chat works for all members
- [ ] Parties can queue for matchmaking as a unit
- [ ] Presence events broadcast on member join/leave
- [ ] Disconnected members are removed after grace period
- [ ] Party dissolves when last member leaves
---
## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Leader disconnect during matchmaking | Orphaned ticket | Medium | Auto-promote leader; pause ticket |
| Party spam (rapid create/destroy) | Server load | Low | Rate limiting on party creation |
| Cross-node parties (scaling) | Complexity | Medium | Party pinned to single node; future: distributed parties |
| Party data abuse (large payloads) | Memory pressure | Low | Data size limits per party |
---
## 10. Future Considerations

- Persistent parties (survive disconnection)
- Party voice chat integration
- Party quests / objectives
- Party XP bonuses
- Cross-server parties (multi-node)
- Party history / statistics
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-08](../PRD/08_parties.md) (API Surface & Interface Specification)
- [TDD-08](../TDD/08_parties.md) (Database Schema & Technical Implementation Design)
