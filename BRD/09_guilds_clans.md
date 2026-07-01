# BRD-09: Guilds & Clans

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Guilds & Clans  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Should Have
---
## 1. Purpose

Define the requirements for a persistent guild/clan system enabling players to form long-lived organizations with membership management, roles, permissions, chat, and metadata. Guilds provide a social structure beyond friends and parties, supporting cooperative gameplay and community building.
---
## 2. Scope

### In Scope
- Guild creation and configuration
- Membership management (join, leave, kick, ban)
- Role system (owner, admin, member, etc.)
- Join request approval workflow
- Guild chat
- Guild metadata and customization
- Guild listing and search
- Member limits

### Out of Scope
- Guild leaderboards / rankings (see BRD-05, BRD-06)
- Guild vs. guild combat mechanics (game-specific)
- Guild treasury / shared economy (future consideration)
- Non-guild groups (see BRD-16: Groups)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Players | Join and participate in guilds |
| Guild Leaders | Manage membership and guild settings |
| Game Designers | Guild-based gameplay features |
| Community Managers | Guild moderation and monitoring |
---
## 4. Functional Requirements

### FR-GD-001: Guild Creation
- **Priority:** Must
- Any player shall create a guild.
- Parameters:
  - `name`: display name (unique, 3-64 characters)
  - `description`: text description (max 1024 characters)
  - `avatar_url`: guild avatar/icon
  - `lang_tag`: primary language
  - `open`: boolean (open = auto-join; closed = request required)
  - `max_count`: maximum members (default: 100, max: configurable)
  - `metadata`: JSON object (max 16 KB)
- The creator becomes the guild owner (superadmin).
- A player can belong to multiple guilds (configurable limit, default: 5).

### FR-GD-002: Membership Roles
- **Priority:** Must
- Role hierarchy:
  - `0` — Superadmin (owner, one per guild)
  - `1` — Admin (can manage members, edit settings)
  - `2` — Member (standard member)
  - `3` — Join request (pending approval)
- Permissions by role:

| Action | Superadmin | Admin | Member | Pending |
|--------|-----------|-------|--------|---------|
| Edit guild settings | ✅ | ✅ | ❌ | ❌ |
| Promote/demote | ✅ | ❌ | ❌ | ❌ |
| Kick members | ✅ | ✅ | ❌ | ❌ |
| Approve requests | ✅ | ✅ | ❌ | ❌ |
| Send chat | ✅ | ✅ | ✅ | ❌ |
| View members | ✅ | ✅ | ✅ | ❌ |

### FR-GD-003: Join Guild
- **Priority:** Must
- **Open guilds**: player joins immediately.
- **Closed guilds**: player sends a join request.
  - Admins/owner approve or reject the request.
  - Requestor is notified of the outcome.
- Join is rejected if guild is full.
- A player cannot join if they are banned from the guild.

### FR-GD-004: Leave Guild
- **Priority:** Must
- Any member can leave a guild voluntarily.
- If the owner leaves:
  - Must transfer ownership first, OR
  - If last member, the guild is dissolved.
- Leaving removes the member from guild chat.

### FR-GD-005: Kick Member
- **Priority:** Must
- Admins and owner can kick members.
- Admins cannot kick other admins or the owner.
- Kicked members can rejoin (unless banned).

### FR-GD-006: Ban Member
- **Priority:** Should
- Owner and admins can ban a user from the guild.
- Banned users cannot rejoin or send join requests.
- Ban list is maintained per guild.
- Unban is supported.

### FR-GD-007: Promote / Demote
- **Priority:** Must
- Owner can promote members to admin or demote admins to member.
- Owner can transfer ownership to another member (owner becomes admin).
- Admins cannot promote/demote other admins.

### FR-GD-008: Guild Chat
- **Priority:** Must
- Each guild shall have a persistent chat channel.
- All members (not pending) can read and send messages.
- Chat history is persisted (configurable retention period).
- Integration with Chat System (BRD-10).

### FR-GD-009: Guild Metadata
- **Priority:** Should
- Guild-level metadata (JSON, max 16 KB):
  - Custom fields: emblem, motto, recruitment status, requirements
  - Updated by admins/owner
- Useful for game-specific guild features (guild level, guild buffs, etc.).

### FR-GD-010: Guild Listing & Search
- **Priority:** Should
- List guilds with filters:
  - Name (partial match)
  - Open/closed
  - Language
  - Member count range
  - Category/tag
- Pagination support.
- Results: guild ID, name, description, member count, open status, metadata.

### FR-GD-011: Guild Deletion
- **Priority:** Must
- Only the owner can delete a guild.
- All members are removed and notified.
- Chat history is archived or deleted (configurable).
- Guild data is permanently removed.

### FR-GD-012: Guild Hooks
- **Priority:** Should
- Server runtime hooks:
  - `before_group_join`: validate/deny join
  - `after_group_join`: side effects
  - `before_group_leave`: validate/deny leave
  - `before_group_update`: validate metadata changes
  - `after_group_update`: side effects
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Guild operations ≤ 100ms (p99) |
| **Scalability** | Support 100,000+ guilds |
| **Capacity** | Support up to 1,000 members per guild (configurable) |
| **Search** | Guild listing ≤ 200ms |
| **Availability** | 99.9% uptime |
| **Data Retention** | Guild chat history retained per configuration |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Member identity |
| BRD-10: Chat System | Guild chat channel |
| BRD-11: Notifications | Guild event notifications |
| BRD-18: Server Runtime & Hooks | Guild lifecycle hooks |
| BRD-12: Storage Engine | Guild-specific data storage |
| PostgreSQL | Guild and membership persistence |
---
## 7. Acceptance Criteria

- [ ] Guilds can be created with name, description, and open/closed setting
- [ ] Open guilds allow immediate join; closed guilds require approval
- [ ] Role hierarchy (owner, admin, member) is enforced for all actions
- [ ] Members can be promoted, demoted, kicked, and banned
- [ ] Guild chat is available to all active members
- [ ] Guilds can be searched by name, language, and open status
- [ ] Member limits are enforced
- [ ] Ownership transfer works correctly
- [ ] Guild deletion removes all memberships and notifies members
- [ ] Server hooks fire on guild operations
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Abandoned guilds (inactive owner) | Dead guilds clutter listings | Medium | Auto-dissolve after N days; admin tools |
| Guild name squatting | Names unavailable | Low | Name release on deletion; name change support |
| Mass kick/ban abuse | Member disruption | Low | Audit logs; rate limiting on admin actions |
| Large guilds (1000+ members) | Performance issues | Low | Paginated member lists; indexed queries |
---
## 9. Future Considerations

- Guild treasury (shared resources/currency)
- Guild levels and XP
- Guild alliances (guild-to-guild relationships)
- Guild wars (inter-guild competition)
- Guild achievements
- Guild activity feed / log
- Guild recruitment board
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-09](../PRD/09_guilds_clans.md) (API Surface & Interface Specification)
- [TDD-09](../TDD/09_guilds_clans.md) (Database Schema & Technical Implementation Design)
