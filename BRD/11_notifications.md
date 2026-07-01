# BRD-11: Notifications

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Notifications  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a server-driven notification system that delivers both real-time and persistent notifications to players. Notifications cover friend requests, tournament rewards, daily bonuses, game events, and custom developer-defined alerts.
---
## 2. Scope

### In Scope
- Real-time notification delivery (WebSocket)
- Persistent notification storage (until read/dismissed)
- System-generated notifications (built-in events)
- Developer-generated notifications (custom, via server runtime)
- Notification listing, read status, and deletion
- Notification codes and categories

### Out of Scope
- Push notifications to mobile devices (APNs, FCM) — future integration point
- Email notifications
- SMS notifications
- In-game UI rendering (client-side concern)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Players | Receive timely alerts about game events |
| Game Designers | Drive engagement through notifications |
| Live Operations | Send event announcements, rewards |
| Game Developers | Custom notification triggers |
---
## 4. Functional Requirements

### FR-NT-001: Notification Structure
- **Priority:** Must
- Each notification shall contain:
  - `id`: unique identifier (UUID)
  - `user_id`: recipient
  - `subject`: short title string
  - `content`: JSON payload (arbitrary data)
  - `code`: integer category code (app-defined)
  - `sender_id`: originating user or system (empty for system notifications)
  - `persistent`: boolean (if true, stored until acknowledged)
  - `create_time`: timestamp

### FR-NT-002: Real-Time Delivery
- **Priority:** Must
- Notifications shall be delivered immediately via WebSocket if the user is online.
- If the user is offline and the notification is persistent, it shall be stored and delivered on next login.
- Non-persistent notifications are lost if the user is offline.

### FR-NT-003: Persistent Notifications
- **Priority:** Must
- Persistent notifications shall be stored in the database.
- Stored until the user explicitly acknowledges (marks as read) or deletes.
- Retrievable via listing API.

### FR-NT-004: System Notifications
- **Priority:** Must
- The server shall automatically generate notifications for built-in events:

| Event | Code | Example Content |
|-------|------|----------------|
| Friend request received | `100` | `"UserX wants to be your friend"` |
| Friend request accepted | `101` | `"UserY accepted your friend request"` |
| Guild join request | `200` | `"UserZ wants to join your guild"` |
| Guild join accepted | `201` | `"You have been accepted into GuildA"` |
| Tournament starting soon | `300` | `"Weekend Arena starts in 1 hour"` |
| Tournament rewards | `301` | `"You placed #5 — 2000 gems awarded"` |
| Party invitation | `400` | `"UserX invited you to their party"` |

### FR-NT-005: Custom Notifications
- **Priority:** Must
- Server runtime code shall send custom notifications.
- Parameters: user_id (or list of user_ids), subject, content, code, persistent flag.
- Use cases:
  - Daily login bonus
  - New event announcement
  - Item received
  - Achievement unlocked
  - Maintenance announcement
- Batch send to multiple users supported.

### FR-NT-006: Notification Listing
- **Priority:** Must
- Retrieve a user's persistent notifications.
- Parameters:
  - `limit` (default: 10, max: 100)
  - `cursor` (pagination)
  - `cacheable_cursor` (for caching)
- Results ordered by create_time (newest first).

### FR-NT-007: Notification Acknowledgment
- **Priority:** Must
- Users shall mark notifications as read/acknowledged.
- Acknowledged notifications can be deleted or retained (configurable).
- Bulk acknowledgment supported.

### FR-NT-008: Notification Deletion
- **Priority:** Must
- Users shall delete individual notifications.
- Bulk delete supported.
- Admins can delete notifications for any user.

### FR-NT-009: Notification Hooks
- **Priority:** Should
- Server runtime hooks:
  - `before_notification_send`: filter/modify/suppress notifications
  - `after_notification_send`: logging, analytics

### FR-NT-010: Broadcast Notifications
- **Priority:** Could
- Send a notification to all online users (broadcast).
- Use cases: server maintenance, global events.
- Must be rate-limited to prevent abuse.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Delivery** | Real-time notifications delivered within 200ms |
| **Storage** | Persistent notifications stored until acknowledged |
| **Scalability** | Support 1,000,000+ stored notifications |
| **Throughput** | Support 10,000+ notification sends per second |
| **Availability** | 99.9% uptime |
| **Retention** | Configurable max age for persistent notifications (default: 30 days) |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Recipient identity |
| BRD-07: Friends System | Friend event notifications |
| BRD-08: Parties | Party invitation notifications |
| BRD-09: Guilds & Clans | Guild event notifications |
| BRD-06: Tournaments | Tournament event notifications |
| BRD-18: Server Runtime & Hooks | Custom notification logic |
| PostgreSQL | Persistent notification storage |
---
## 7. Acceptance Criteria

- [ ] Real-time notifications are delivered via WebSocket when user is online
- [ ] Persistent notifications are stored and delivered on next login
- [ ] System notifications fire for built-in events (friends, guilds, tournaments)
- [ ] Custom notifications can be sent from server runtime code
- [ ] Notification listing supports pagination
- [ ] Notifications can be acknowledged and deleted
- [ ] Batch send works for multiple recipients
- [ ] Notification hooks can filter/suppress before delivery
- [ ] Non-persistent notifications are discarded when user is offline
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Notification spam | Player annoyance, churn | Medium | Rate limiting; categories; mute options |
| Storage growth (unread notifications) | Database bloat | Medium | TTL on persistent notifications; max count per user |
| Delivery failures (WebSocket disconnect) | Missed real-time alerts | Low | Fallback to persistent storage; retry on reconnect |
| Broadcast abuse | Server overload | Low | Rate limiting; admin-only broadcast |
---
## 9. Future Considerations

- Push notification integration (APNs, FCM, WebPush)
- Email notification integration
- Notification preferences (opt-in/out per category)
- Notification scheduling (send at specific time)
- Rich notifications (images, actions, deep links)
- Notification analytics (open rates, engagement)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-11](../PRD/11_notifications.md) (API Surface & Interface Specification)
- [TDD-11](../TDD/11_notifications.md) (Database Schema & Technical Implementation Design)
