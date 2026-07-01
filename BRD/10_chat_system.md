# BRD-10: Chat System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Chat System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a comprehensive real-time chat system supporting direct messages, group channels, guild channels, party channels, match channels, and custom global channels. The system supports persistent history, moderation, and presence tracking.
---
## 2. Scope

### In Scope
- Direct messaging (1-to-1)
- Group chat channels
- Guild/clan chat channels
- Party chat channels
- Match chat channels
- Custom/global channels
- Persistent message history
- Moderation hooks
- Presence within channels
- Message format (text, metadata)

### Out of Scope
- Voice/video chat
- Rich media attachments (images, files)
- End-to-end encryption
- Chat bots (can be built via server runtime)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Players | Communicate with friends, guild, team |
| Game Developers | Flexible chat integration |
| Community Managers | Moderation and content filtering |
| Trust & Safety | Anti-harassment, spam prevention |
---
## 4. Functional Requirements

### FR-CH-001: Channel Types
- **Priority:** Must
- The system shall support the following channel types:

| Type | Code | Description | Persistence |
|------|------|-------------|-------------|
| Direct Message | `1` | 1-to-1 between two users | Persistent |
| Group | `2` | Named multi-user channel | Persistent |
| Guild | `3` | Guild-specific channel | Persistent |
| Party | `4` | Party-specific channel | Ephemeral |
| Match | `5` | Match-specific channel | Ephemeral |
| Custom | `6` | Developer-defined global channel | Configurable |

### FR-CH-002: Direct Messages
- **Priority:** Must
- Players shall send direct messages to other players.
- Blocked users cannot send DMs (see BRD-07).
- DM channels are auto-created on first message.
- DM history is persistent and queryable.
- Both participants must be able to access the history.

### FR-CH-003: Group Channels
- **Priority:** Should
- Named channels that players join and leave.
- Configurable: open (anyone joins) or invite-only.
- Support for channel owner/admin roles.
- Persistent history.

### FR-CH-004: Guild Chat
- **Priority:** Must
- Each guild automatically has a chat channel.
- Only guild members can read/write.
- Channel is created/destroyed with the guild.
- Persistent history (retention configurable).
- See BRD-09 for guild membership.

### FR-CH-005: Party Chat
- **Priority:** Must
- Each party automatically has a chat channel.
- Only party members can read/write.
- Channel is ephemeral — destroyed when party dissolves.
- No persistent history.

### FR-CH-006: Match Chat
- **Priority:** Should
- Each match can have an associated chat channel.
- Only match participants can read/write.
- Channel is ephemeral — destroyed when match ends.
- Configurable: team chat (subsets) vs. all-chat.

### FR-CH-007: Send Message
- **Priority:** Must
- Users send messages to a channel.
- Message structure:
  - `message_id`: UUID
  - `channel_id`: target channel
  - `sender_id`: user ID
  - `content`: JSON object (`{"text": "...", ...}`)
  - `code`: integer message type code (app-defined)
  - `create_time`: timestamp
- Messages are delivered in real-time via WebSocket to all channel members.
- Messages are stored in persistent channels.

### FR-CH-008: Message History
- **Priority:** Must
- Persistent channels retain message history.
- Retrieve history with:
  - `channel_id`
  - `limit` (default: 20, max: 100)
  - `cursor` (pagination, forward/backward)
- Messages returned in chronological order.
- Configurable retention period (default: 30 days).

### FR-CH-009: Channel Presence
- **Priority:** Should
- Track which users are currently in a channel.
- Presence events: user joined channel, user left channel.
- Typing indicators (optional, via custom messages).

### FR-CH-010: Moderation
- **Priority:** Should
- Server runtime hooks for moderation:
  - `before_channel_message_send`: filter/block messages before delivery
  - `after_channel_message_send`: logging, analytics
- Support for:
  - Word filtering (configurable blocklist)
  - Spam detection (rate limiting)
  - User muting within channels
  - Message deletion by admins

### FR-CH-011: Channel Join / Leave
- **Priority:** Must
- Users join a channel to receive messages.
- Users leave a channel to stop receiving messages.
- Auto-join behavior for guild, party, match channels.
- Manual join for group and custom channels.

### FR-CH-012: Message Update / Delete
- **Priority:** Should
- Users can edit their own messages (within a time window).
- Users can delete their own messages.
- Admins can delete any message.
- Edit/delete events are broadcast to channel members.

### FR-CH-013: Unread Tracking
- **Priority:** Could
- Track last read position per user per channel.
- Calculate unread count for each channel.
- Mark channel as read.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | Message delivery ≤ 100ms (server-to-client) |
| **Throughput** | Support 10,000+ messages per second (global) |
| **Scalability** | Support 100,000+ active channels |
| **Storage** | Efficient message storage with configurable retention |
| **Availability** | 99.9% uptime |
| **Content Size** | Max message content: 4 KB |
| **History** | History queries ≤ 100ms |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Sender identity |
| BRD-07: Friends System | Block list enforcement for DMs |
| BRD-08: Parties | Party chat channels |
| BRD-09: Guilds & Clans | Guild chat channels |
| BRD-03: Realtime Multiplayer | Match chat channels |
| BRD-18: Server Runtime & Hooks | Moderation hooks |
| PostgreSQL | Persistent message storage |
---
## 7. Acceptance Criteria

- [ ] Direct messages can be sent between two users
- [ ] Guild and party channels are auto-created with their parent entities
- [ ] Messages are delivered in real-time via WebSocket
- [ ] Persistent channels retain message history with pagination
- [ ] Blocked users cannot send DMs
- [ ] Moderation hooks can filter/block messages before delivery
- [ ] Channel presence events (join/leave) are broadcast
- [ ] Messages can be edited and deleted
- [ ] Message delivery latency ≤ 100ms
- [ ] Max message size (4 KB) is enforced
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Chat spam / flooding | Player disruption | High | Rate limiting; moderation hooks; muting |
| Toxic content | Community harm | High | Content filters; reporting; hooks |
| Message storage growth | Database bloat | Medium | Configurable retention; TTL; archival |
| Large channel member count | Performance | Low | Fan-out optimization; message queuing |
---
## 9. Future Considerations

- Rich media messages (images, links with preview)
- Chat reactions (emoji reactions on messages)
- Threaded replies
- Message search
- Chat bots / automated responses
- Localization support (auto-translate)
- End-to-end encryption for DMs
- Offline message delivery (push notification integration)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-10](../PRD/10_chat_system.md) (API Surface & Interface Specification)
- [TDD-10](../TDD/10_chat_system.md) (Database Schema & Technical Implementation Design)
