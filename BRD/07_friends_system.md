# BRD-07: Friends System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Friends System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a built-in social graph enabling players to add friends, manage friend requests, block users, and track online status. The friends system is foundational to social features like party formation, direct messaging, and friend leaderboards.
---
## 2. Scope

### In Scope
- Friend requests (send, accept, reject)
- Friend listing with state filters
- Unfriending
- Blocking and unblocking users
- Online status / presence of friends
- Friend count limits
- Mutual vs. one-directional relationships

### Out of Scope
- Direct messaging (see BRD-10: Chat System)
- Party formation (see BRD-08: Parties)
- Social media integration for friend import
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Players | Connect with friends, see online status |
| Game Designers | Social engagement, friend-based features |
| Game Developers | Simple friend API, presence integration |
| Trust & Safety | Block/mute functionality |
---
## 4. Functional Requirements

### FR-FR-001: Send Friend Request
- **Priority:** Must
- A player shall send a friend request to another player by user ID or username.
- The target player shall receive a notification.
- Duplicate requests to the same user shall be idempotent.
- Blocked users cannot send friend requests.

### FR-FR-002: Accept Friend Request
- **Priority:** Must
- The recipient shall accept a pending friend request.
- On acceptance, both users become mutual friends (state = `friends`).
- Both users shall receive a notification.

### FR-FR-003: Reject Friend Request
- **Priority:** Must
- The recipient shall reject a pending friend request.
- The request is removed from the pending list.
- The sender is not notified of rejection (privacy).

### FR-FR-004: List Friends
- **Priority:** Must
- Retrieve the list of a player's friends with state filter.
- Friend states:
  - `0` — Mutual friends
  - `1` — Outgoing friend request (pending, sent by this user)
  - `2` — Incoming friend request (pending, sent to this user)
  - `3` — Blocked
- Pagination support (cursor-based).
- Each friend entry includes: user ID, username, display name, avatar, state, online status, update time.

### FR-FR-005: Remove Friend
- **Priority:** Must
- A player shall remove a mutual friend.
- Removal is bidirectional (both users lose the friendship).
- No notification on removal (configurable via hook).

### FR-FR-006: Block User
- **Priority:** Must
- A player shall block another user.
- Blocking shall:
  - Remove existing friendship (if any)
  - Prevent the blocked user from sending friend requests
  - Hide the blocker's online status from the blocked user
  - Prevent direct messages from the blocked user
- Blocking is one-directional (A blocks B; B doesn't know).

### FR-FR-007: Unblock User
- **Priority:** Must
- A player shall unblock a previously blocked user.
- Unblocking does not restore the previous friendship.
- The unblocked user can now send friend requests again.

### FR-FR-008: Friend Count Limits
- **Priority:** Should
- Maximum friend count per user: configurable (default: 1,000).
- Maximum pending requests: configurable (default: 100).
- When limits are reached, new requests are rejected with an error.

### FR-FR-009: Friend Presence
- **Priority:** Must
- Players shall see which friends are currently online.
- Presence data includes: online status, current activity (match, menu, etc.).
- Presence updates shall be delivered in real-time via WebSocket.
- Integration with the Presence System (BRD-17).

### FR-FR-010: Import Social Friends
- **Priority:** Could
- Import friends from social platforms (Facebook, Steam, etc.).
- Match imported contacts against existing users.
- Automatically send friend requests to matched users.
- Requires user consent.

### FR-FR-011: Friend Hooks
- **Priority:** Should
- Server runtime hooks:
  - `before_add_friends`: validate/deny friend request
  - `after_add_friends`: side effects (achievement tracking)
  - `before_block_friends`: validate/deny block
  - `after_block_friends`: side effects
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Friend list retrieval ≤ 100ms |
| **Scalability** | Support millions of friend relationships |
| **Consistency** | Friend state changes reflected immediately |
| **Privacy** | Blocked users cannot detect they are blocked |
| **Availability** | 99.9% uptime |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | User identity |
| BRD-17: Presence System | Online status tracking |
| BRD-11: Notifications | Friend request notifications |
| BRD-10: Chat System | Direct messaging between friends |
| BRD-18: Server Runtime & Hooks | Friend event hooks |
---
## 7. Acceptance Criteria

- [ ] Players can send friend requests by user ID or username
- [ ] Friend requests can be accepted or rejected
- [ ] Mutual friends are correctly established on acceptance
- [ ] Friends can be removed (bidirectional)
- [ ] Blocking prevents all interaction and hides status
- [ ] Unblocking allows future interaction
- [ ] Friend list retrieval supports state filtering and pagination
- [ ] Online status of friends is visible in real-time
- [ ] Friend count limits are enforced
- [ ] Server hooks fire on friend operations
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Friend spam (mass requests) | Player harassment | Medium | Rate limiting on friend requests; max pending limit |
| Privacy concerns (online status) | Trust issues | Medium | Configurable visibility; invisible mode |
| Large friend lists (performance) | Slow queries | Low | Pagination; indexed queries; limit enforcement |
| Social graph data leak | Privacy violation | Low | Strict permission model; block enforcement |
---
## 9. Future Considerations

- Friend suggestions (based on play history, mutual friends)
- Favorite friends (pinned friends for quick access)
- Friend categories / groups
- Invisible / appear offline mode
- Friend activity feed
- Cross-platform friend linking
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-07](../PRD/07_friends_system.md) (API Surface & Interface Specification)
- [TDD-07](../TDD/07_friends_system.md) (Database Schema & Technical Implementation Design)
