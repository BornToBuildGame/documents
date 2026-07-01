# BRD-02: Multiplayer Matchmaking

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Multiplayer Matchmaking  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a flexible matchmaking system that pairs players into compatible game sessions based on skill, region, game mode, and custom properties. The system shall support solo players, parties, and custom matchmaking logic.
---
## 2. Scope

### In Scope
- Automatic matchmaking with configurable criteria
- Skill-based matchmaking (MMR / Elo)
- Region-based filtering
- Custom matchmaking properties
- Party matchmaking (group queue)
- Matchmaking ticket management
- Match found notifications
- Count Multiple support (team-size/player multiplier constraints)
- Reverse Matching (bidirectional validation)
- Offline matchmaking support (bots, offline opponents)

### Out of Scope
- Game session hosting (see BRD-03: Realtime Multiplayer)
- Game logic execution (see BRD-04: Authoritative Game Server)
- Leaderboard ranking calculations (see BRD-05: Leaderboards)
- Push notifications for offline users (see BRD-11)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Configurable matchmaking logic, easy integration |
| Players | Fair matches, fast queue times |
| Game Designers | Skill-based pairing, mode-specific rules |
| Server Administrators | Queue monitoring, performance tuning |
---
## 4. Functional Requirements

### FR-MM-001: Matchmaking Ticket Submission
- **Priority:** Must
- Players shall submit a matchmaking ticket to join a queue.
- A ticket shall contain:
  - Player ID (or party ID)
  - Queue name (game mode identifier)
  - Matchmaking properties (JSON)
  - Optional: min/max player count preferences
- A player shall only hold one active ticket at a time.
- Submitting a new ticket shall cancel any existing ticket.

### FR-MM-002: Automatic Matchmaking
- **Priority:** Must
- The system shall automatically group compatible players into a match.
- Match formation shall occur when enough compatible players are found.
- Configurable match size: minimum and maximum players per match.
- Timeout: if no match is found within a configurable duration, the ticket expires.

### FR-MM-003: Skill-Based Matchmaking
- **Priority:** Must
- Players shall be matched based on a numeric skill rating (MMR).
- The system shall define a skill window: initially narrow, widening over time.
- Configuration per queue:
  - `initial_skill_range`: starting MMR window (e.g., ±50)
  - `max_skill_range`: maximum MMR window (e.g., ±500)
  - `expansion_interval`: time between window expansions (e.g., 5 seconds)
  - `expansion_step`: MMR increase per expansion (e.g., +25)

### FR-MM-004: Region Filtering
- **Priority:** Should
- Players shall specify a preferred region (e.g., `us-east`, `eu-west`, `asia`).
- The matchmaker shall prefer same-region matches.
- Configuration:
  - `strict_region`: if true, only match within the same region
  - `region_fallback_delay`: time before expanding to other regions

### FR-MM-005: Custom Matchmaking Properties
- **Priority:** Must
- Tickets shall carry arbitrary key-value properties.
- The matchmaker shall support filtering on custom properties.
- Examples:
  - `game_mode: "ranked"` or `"casual"`
  - `map: "dust2"`
  - `character_level_range: [10, 20]`
- Property matching shall support: exact match, range match, and set membership.

### FR-MM-006: Party Matchmaking
- **Priority:** Must
- Parties shall be able to queue as a unit.
- All party members join and leave the queue together.
- The party leader submits the matchmaking ticket.
- Party skill rating: configurable (average, highest, or weighted).
- Party size constraints shall be enforceable per queue.

### FR-MM-007: Ticket Cancellation
- **Priority:** Must
- Players/parties shall be able to cancel their matchmaking ticket.
- Disconnected players' tickets shall be automatically cancelled after a timeout.
- Party ticket cancellation shall remove the entire party from the queue.

### FR-MM-008: Match Found Notification
- **Priority:** Must
- When a match is formed, all matched players shall receive a notification.
- Notification shall include:
  - Match ID
  - Match token
  - List of players (IDs, usernames)
  - Server connection info (if applicable)
- Delivery via WebSocket (realtime) and/or polling endpoint.

### FR-MM-009: Custom Matchmaker Logic
- **Priority:** Should
- Developers shall be able to override or extend the default matchmaker.
- Server runtime hooks:
  - `matchmaker_matched`: intercept match formation to accept/reject/modify
  - Custom scoring functions
- Support writing custom matchmaker logic in the server runtime (Lua/TypeScript/Go).

### FR-MM-010: Queue Management
- **Priority:** Must
- Support multiple named queues (e.g., `ranked_1v1`, `casual_5v5`, `custom`).
- Each queue shall have independent configuration:
  - Min/max players
  - Skill range
  - Region rules
  - Custom property requirements
  - Timeout duration

### FR-MM-011: Backfill Support
- **Priority:** Could
- Active matches shall be able to request backfill (find replacement players).
- Backfill tickets shall have higher priority in the queue.
- Replacement players shall meet the same matchmaking criteria.

### FR-MM-012: Count Multiple
- **Priority:** Should
- The matchmaking system shall support a `count_multiple` parameter in tickets.
- If specified, the matchmaker shall only form matches whose final player size is an exact integer multiple of the specified count multiple.
- Used for: team size multipliers (e.g., in a 2v2v2v2 mode, `count_multiple` would be 2 or 4).

### FR-MM-013: Reverse Matching (Bidirectional Validation)
- **Priority:** Should
- The matchmaker shall support bidirectional validation (`reverse_precision`) configurations.
- When enabled, the matchmaker validates that not only does Player A match Player B's criteria, but Player B's criteria also matches Player A.
- A configurable `reverse_threshold` interval determines how many matchmaking ticks bidirectional matching is enforced before falling back to unidirectional (one-way matching) to prevent player starvation.

### FR-MM-014: Offline Matchmaking
- **Priority:** Should
- The system shall support offline matchmaking patterns, allowing developers to populate matchmaking queues with offline player records or bots.
- Developers shall be able to add, manage, and query tickets for offline/system-owned participants.
- Custom matchmaking hooks shall be able to authoritatively inject bot players or pull offline player snapshots to complete matches when active queues are low.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Average match formation time ≤ 30 seconds for populated queues |
| **Scalability** | Support 100,000+ concurrent tickets across all queues |
| **Fairness** | Matched players' MMR shall be within configured ranges |
| **Availability** | 99.9% uptime for matchmaking service |
| **Latency** | Ticket submission acknowledged within 100ms |
| **Monitoring** | Queue depth, average wait time, match formation rate metrics |
| **Elasticity** | Queue processing shall scale with ticket volume |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Player identity and sessions |
| BRD-08: Parties | Party matchmaking integration |
| BRD-03: Realtime Multiplayer | Match session creation after matchmaking |
| BRD-17: Presence System | Player online status for matchmaking |
| PostgreSQL | Ticket and match metadata persistence |
---
## 7. Acceptance Criteria

- [ ] Players can submit matchmaking tickets with skill rating and properties
- [ ] The matchmaker forms matches with compatible players within skill windows
- [ ] Skill windows expand over time to reduce wait times
- [ ] Region filtering works with configurable strict/fallback modes
- [ ] Parties can queue together as a unit
- [ ] Tickets can be cancelled by the player or automatically on disconnect
- [ ] Match found notifications are delivered in real time via WebSocket
- [ ] Multiple independent queues can be configured
- [ ] Custom matchmaker hooks can accept/reject/modify match formation
- [ ] Average queue time ≤ 30s for populated queues
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Long queue times (low population) | Poor player experience | High (early) | Aggressive skill window expansion; bot backfill |
| Skill disparity in matches | Unfair games | Medium | Tunable skill windows; post-match MMR adjustment |
| Party boosting (high + low skill) | Ranking manipulation | Medium | Use highest party member MMR; limit skill variance |
| Queue starvation (niche modes) | Dead queues | Medium | Merge queues dynamically; show queue population |
| Matchmaker overload | Increased latency | Low | Horizontal scaling; queue sharding |
---
## 9. Future Considerations

- Machine learning-based matchmaking (predict match quality)
- Cross-play matchmaking (platform-aware matching)
- Ranked seasons with placement matches
- Anti-smurf detection
- Queue dodge penalties
- Predicted wait time display
- Server-side ping estimation for region selection
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-02](../PRD/02_multiplayer_matchmaking.md) (API Surface & Interface Specification)
- [TDD-02](../TDD/02_multiplayer_matchmaking.md) (Database Schema & Technical Implementation Design)
