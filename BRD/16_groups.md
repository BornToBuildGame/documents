# BRD-16: Groups

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Groups  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Should Have
---
## 1. Purpose

Define the requirements for a general-purpose group system, distinct from guilds/clans, for organizing players into communities, clubs, schools, teams, and other non-competitive associations. Groups share the same underlying data model as guilds but serve different use cases.
---
## 2. Scope

### In Scope
- Group creation for communities, clubs, teams, etc.
- Membership management (join, leave, roles)
- Group types and categories
- Group metadata
- Group listing and search
- Permissions model
- Group chat integration

### Out of Scope
- Guild-specific features (guild wars, guild treasury — see BRD-09)
- Competitive group features
- Group matchmaking
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Players | Join communities, clubs, study groups |
| Community Managers | Organize and moderate groups |
| Game Developers | Flexible group API |
| Educators / Coaches | Team and class management |
---
## 4. Functional Requirements

### FR-GR-001: Group Types
- **Priority:** Must
- Groups shall be categorized by type using integer categories.
- Suggested categories (developer-configurable):

| Category | Code | Use Case |
|----------|------|----------|
| Guild | `0` | Game guilds/clans |
| Community | `1` | General communities |
| Club | `2` | Interest-based clubs |
| School/Class | `3` | Educational groups |
| Team | `4` | Competitive teams |
| Custom | `5+` | Developer-defined |

- The groups API is shared for all types (see BRD-09 for guild-specific BRD).
- Category is used for filtering and listing.

### FR-GR-002: Group Creation
- **Priority:** Must
- Same creation flow as guilds (uses the same `groups` table).
- Parameters: name, description, category, open/closed, max_count, metadata.
- The category field distinguishes groups from guilds.
- Users can belong to multiple groups (configurable limit).

### FR-GR-003: Membership Management
- **Priority:** Must
- Identical to guild membership (see BRD-09 FR-GD-002 through FR-GD-007):
  - Join / Leave
  - Invite / Kick / Ban
  - Promote / Demote
  - Role hierarchy (superadmin, admin, member, join_request)

### FR-GR-004: Group Permissions
- **Priority:** Should
- Configurable per-group permissions:
  - Who can invite (owner only, admins, any member)
  - Who can post chat (all members, admins only)
  - Who can view member list (all, members only, public)
- Stored in group metadata.

### FR-GR-005: Group Discovery
- **Priority:** Must
- Search and filter groups:
  - By category
  - By name (partial match)
  - By open/closed
  - By language
  - By member count range
- Results exclude groups the user is already in (optional).
- Pagination support.

### FR-GR-006: Group Chat
- **Priority:** Should
- Each group has an associated chat channel (same as guilds).
- Chat access follows group membership and permission settings.
- Integration with Chat System (BRD-10).

### FR-GR-007: User's Groups
- **Priority:** Must
- List all groups a user belongs to.
- Filter by category, state (member, admin, pending).
- Pagination support.

### FR-GR-008: Group Limits
- **Priority:** Should
- Max groups per user: configurable (default: 10, includes guilds).
- Max members per group: configurable per group at creation.
- Limits enforced on join operations.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Group operations ≤ 100ms (p99) |
| **Scalability** | Support 500,000+ groups |
| **Search** | Group listing ≤ 200ms |
| **Availability** | 99.9% uptime |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-09: Guilds & Clans | Shared data model and API |
| BRD-01: User Authentication | Member identity |
| BRD-10: Chat System | Group chat channels |
| BRD-11: Notifications | Group event notifications |
| PostgreSQL | Group persistence |
---
## 7. Acceptance Criteria

- [ ] Groups can be created with non-guild categories (community, club, team, etc.)
- [ ] Groups are distinguishable from guilds by category
- [ ] Group search filters by category
- [ ] Membership management works identically to guilds
- [ ] Users can belong to multiple groups
- [ ] Group limits (per user, per group) are enforced
- [ ] Group chat is available for group members
- [ ] Group listing returns correctly filtered results
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Confusion between groups and guilds | Developer confusion | Medium | Clear documentation; category naming conventions |
| Shared API limitations | Feature constraints | Low | Category-based hooks for different behavior |
| Group proliferation | Database growth | Low | Group limits per user; inactive group cleanup |
---
## 9. Future Considerations

- Group templates (pre-configured group types)
- Sub-groups (hierarchical groups)
- Group events / calendar
- Group activity feed
- Group discovery recommendations (based on interests, play style)
- Cross-group messaging
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-16](../PRD/16_groups.md) (API Surface & Interface Specification)
- [TDD-16](../TDD/16_groups.md) (Database Schema & Technical Implementation Design)
