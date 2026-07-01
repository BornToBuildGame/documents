# BRD-06: Tournaments

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Tournaments  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Should Have
---
## 1. Purpose

Define the requirements for a tournament system that supports scheduled competitive events with entry limits, durations, multiple seasons, and rewards. Tournaments build on the leaderboard system to provide time-bounded competitive experiences.
---
## 2. Scope

### In Scope
- Tournament creation and configuration
- Scheduled start and end times
- Entry limits and join conditions
- Score submission within tournament windows
- Season management (recurring tournaments)
- Rewards distribution (via hooks)
- Tournament listing and discovery

### Out of Scope
- Bracket / elimination tournament formats (future consideration)
- Reward item definitions (see BRD-13: Economy System)
- Matchmaking integration (see BRD-02)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Designers | Create engaging competitive events |
| Players | Compete in timed events, earn rewards |
| Live Operations | Schedule and manage tournaments |
| Game Developers | Integrate tournament flow into game |
---
## 4. Functional Requirements

### FR-TN-001: Tournament Creation
- **Priority:** Must
- Tournaments shall be created via API or server configuration.
- Configuration:
  - `tournament_id`: unique identifier
  - `title`: display name
  - `description`: event description
  - `category`: integer category tag
  - `sort_order`: ascending or descending
  - `operator`: best, set, or increment
  - `duration`: active duration in seconds
  - `reset_schedule`: cron expression for recurring tournaments
  - `start_time`: UTC timestamp for first occurrence
  - `end_time`: UTC timestamp for final end (optional)
  - `max_size`: maximum participants (0 = unlimited)
  - `max_num_score`: max score submissions per participant
  - `metadata`: JSON object
  - `authoritative`: server-only score writes

### FR-TN-002: Tournament Scheduling
- **Priority:** Must
- Tournaments shall support:
  - **One-time**: runs once between start_time and start_time + duration
  - **Recurring**: repeats based on reset_schedule (daily, weekly, etc.)
  - **Seasonal**: multiple occurrences with a final end_time
- Tournament states:
  - `upcoming`: before start_time
  - `active`: currently running
  - `ended`: past end_time or duration elapsed
  - `archived`: historical data retained

### FR-TN-003: Tournament Join
- **Priority:** Must
- Players shall explicitly join a tournament before submitting scores.
- Join conditions:
  - Tournament must be active
  - Player not already joined
  - Entry limit not reached (if max_size set)
  - Custom join conditions via hooks (level requirement, entry fee, etc.)
- On join: create a record with initial score (0).

### FR-TN-004: Score Submission
- **Priority:** Must
- Participants submit scores during the active window.
- Score processing follows the configured operator (best/set/incr).
- Submissions after the tournament ends shall be rejected.
- `max_num_score` limits total submissions per participant per occurrence.
- If `authoritative`, reject client-submitted scores.

### FR-TN-005: Rankings
- **Priority:** Must
- Rankings are calculated identically to leaderboards (dense rank).
- Retrieve rankings with pagination.
- Around-player lookup supported.
- Rankings update in real-time as scores are submitted.

### FR-TN-006: Tournament Listing
- **Priority:** Must
- List available tournaments with filtering:
  - Category
  - Active/upcoming/ended
  - Pagination
- Return: tournament ID, title, category, start/end times, current size, max size.

### FR-TN-007: Rewards Distribution
- **Priority:** Should
- When a tournament ends, a server hook shall fire with final rankings.
- The hook can distribute rewards:
  - Currency (wallet updates)
  - Items (storage writes)
  - Notifications
- Reward tiers are configurable per tournament (e.g., top 10, top 100, all participants).
- Rewards are defined in tournament metadata or server-side logic.

### FR-TN-008: Season Management
- **Priority:** Should
- Recurring tournaments accumulate season data.
- Per-season tracking:
  - Total participations
  - Best score
  - Total rewards earned
- Season reset clears active records but retains historical data.
- Season archives accessible via API.

### FR-TN-009: Tournament Hooks
- **Priority:** Should
- Server runtime hooks:
  - `before_tournament_join`: validate/deny join attempt
  - `after_tournament_join`: side effects on join
  - `before_tournament_write`: validate score submission
  - `after_tournament_write`: side effects on score write
  - `on_tournament_end`: final rankings, reward distribution
  - `on_tournament_reset`: season transition
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Score submission ≤ 50ms (p99) |
| **Scalability** | Support 500+ concurrent tournaments |
| **Capacity** | Support 100,000+ participants per tournament |
| **Scheduling** | Tournament state transitions within 1 second of scheduled time |
| **Availability** | 99.9% uptime |
| **Data Retention** | Historical tournament data retained for configurable duration |
---
## 6. Example: Weekend Arena Tournament

```
Tournament: Weekend Arena
├── ID: "weekend_arena"
├── Start: Friday 18:00 UTC
├── Duration: 53h 59m (until Sunday 23:59 UTC)
├── Reset: Weekly (cron: "0 18 * * 5")
├── Max Size: 10,000
├── Operator: best
├── Sort: descending
└── Rewards:
    ├── Rank 1:     5000 gems + Legendary Chest
    ├── Rank 2-10:  2000 gems + Epic Chest
    ├── Rank 11-100: 500 gems + Rare Chest
    └── All others:  100 gems
```
---
## 7. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-05: Leaderboards | Tournaments extend the leaderboard data model |
| BRD-01: User Authentication | Player identity |
| BRD-13: Economy System | Reward distribution (wallet, items) |
| BRD-11: Notifications | Notify players of tournament events |
| BRD-18: Server Runtime & Hooks | Tournament lifecycle hooks |
---
## 8. Acceptance Criteria

- [ ] Tournaments can be created with scheduled start, duration, and reset
- [ ] Players can join tournaments within the active window
- [ ] Scores are submitted and ranked correctly
- [ ] Max participant limit is enforced
- [ ] Tournament state transitions on schedule (upcoming → active → ended)
- [ ] Recurring tournaments reset and start new occurrences
- [ ] End-of-tournament hooks fire with final rankings
- [ ] Tournament listing supports filtering by category and state
- [ ] Historical tournament data is retained and queryable
---
## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Rewards duplication on end hook | Economy exploit | Medium | Idempotent reward distribution; transaction guard |
| Clock skew causing early/late transitions | Wrong tournament state | Low | NTP synchronization; UTC-only scheduling |
| Mass join at tournament start | Spike load | Medium | Rate limiting; pre-registration; queue joining |
| Large tournament rankings query | Slow responses | Medium | Cached rankings; paginated queries |
---
## 10. Future Considerations

- Bracket/elimination tournament format
- Team-based tournaments (guilds compete)
- Entry fees (virtual currency cost to join)
- Qualification rounds
- Live tournament spectating
- Tournament replay highlights
- Cross-server tournaments (multi-region)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-06](../PRD/06_tournaments.md) (API Surface & Interface Specification)
- [TDD-06](../TDD/06_tournaments.md) (Database Schema & Technical Implementation Design)
