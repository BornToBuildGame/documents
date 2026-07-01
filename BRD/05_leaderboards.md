# BRD-05: Leaderboards

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Leaderboards  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a built-in leaderboard system supporting multiple timeframes, ranking strategies, pagination, and around-player lookups. Leaderboards drive competitive engagement and provide visible progression for players.
---
## 2. Scope

### In Scope
- Leaderboard creation and configuration
- Score submission and aggregation
- Multiple ranking strategies (best, set, increment)
- Time-based leaderboards (daily, weekly, monthly, seasonal, all-time)
- Automated reset schedules
- Pagination and around-player lookups
- Per-record metadata
- Leaderboard deletion and archival
- Bucketed/bracketed leaderboards (partitioning large leaderboards into brackets)

### Out of Scope
- Tournament logic (see BRD-06: Tournaments)
- MMR/Elo calculation algorithms (game-specific)
- Rewards distribution (handled by economy/tournament systems)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Designers | Define competitive ranking systems |
| Players | See their ranking, compare with others |
| Game Developers | Easy score submission, flexible queries |
| Community Managers | Monitor rankings, detect anomalies |
---
## 4. Functional Requirements

### FR-LB-001: Leaderboard Creation
- **Priority:** Must
- Leaderboards shall be created via API or server configuration.
- Configuration parameters:
  - `leaderboard_id`: unique string identifier
  - `sort_order`: `ascending` or `descending`
  - `operator`: `best` (keep highest/lowest), `set` (overwrite), `incr` (accumulate)
  - `reset_schedule`: cron expression (e.g., `0 0 * * 1` for weekly Monday reset)
  - `metadata`: JSON object for leaderboard-level metadata
  - `authoritative`: if true, only server-side code can write scores

### FR-LB-002: Score Submission
- **Priority:** Must
- Players shall submit scores to a leaderboard.
- Submission includes:
  - `leaderboard_id`
  - `owner_id` (user ID)
  - `score`: int64
  - `subscore`: int64 (secondary sort)
  - `metadata`: JSON (per-record, e.g., character used, match details)
- Score processing based on operator:
  - `best`: only update if new score is better than existing
  - `set`: always overwrite
  - `incr`: add to existing score
- If `authoritative`, reject client-submitted scores.

### FR-LB-003: Ranking
- **Priority:** Must
- Each record shall have a `rank` field calculated from position in the leaderboard.
- Ranking shall be dense rank (no gaps after ties) or configurable.
- Tied scores shall be sorted by subscore, then by earliest submission.

### FR-LB-004: Leaderboard Listing
- **Priority:** Must
- Retrieve leaderboard records with pagination.
- Parameters:
  - `leaderboard_id`
  - `limit` (default: 10, max: 100)
  - `cursor` (pagination token)
- Return: list of records with rank, owner ID, username, score, subscore, metadata, update time.

### FR-LB-005: Around-Player Lookup
- **Priority:** Must
- Retrieve records surrounding a specific player's rank.
- Parameters:
  - `leaderboard_id`
  - `owner_id`
  - `limit` (number of records above and below)
- Return: centered window of records around the player.

### FR-LB-006: Player Record Lookup
- **Priority:** Must
- Retrieve a specific player's record from a leaderboard.
- Retrieve multiple players' records in a batch.
- Return includes the player's current rank.

### FR-LB-007: Reset Schedules
- **Priority:** Must
- Leaderboards shall support automated resets based on cron schedules.
- Common presets:
  - Daily: `0 0 * * *`
  - Weekly: `0 0 * * 1`
  - Monthly: `0 0 1 * *`
- On reset:
  - Current records are archived (optional, configurable)
  - Active records are cleared
  - Reset event is emitted (hookable)
- Manual reset via admin API.

### FR-LB-008: Time-Based Leaderboards
- **Priority:** Must
- Support predefined timeframes:
  - **All-time**: never resets
  - **Daily**: resets every 24 hours
  - **Weekly**: resets every 7 days
  - **Monthly**: resets every month
  - **Seasonal**: custom reset schedule (e.g., every 3 months)
- Each timeframe can have independent leaderboard instances.

### FR-LB-009: Leaderboard Metadata
- **Priority:** Should
- Leaderboard-level metadata (JSON) for display purposes.
- Per-record metadata (JSON) for contextual information.
- Max metadata size: 16 KB per record.

### FR-LB-010: Leaderboard Deletion
- **Priority:** Should
- Support deleting a leaderboard and all its records.
- Support deleting individual records.
- Support archiving before deletion.

### FR-LB-011: Leaderboard Hooks
- **Priority:** Should
- Server runtime hooks for leaderboard events:
  - `before_leaderboard_record_write`: validate/modify before writing
  - `after_leaderboard_record_write`: trigger side effects
  - `on_leaderboard_reset`: post-reset processing (rewards, archival)

### FR-LB-012: Bucketed Leaderboards
- **Priority:** Should
- The system shall support partitioning massive leaderboards into smaller, isolated "buckets" or "brackets" of a configurable size (e.g., 50 or 100 players each).
- Players are dynamically or statically assigned to a bucket upon their first score submission or tournament entry.
- Leaderboard queries (retrieval, around-player) within a bucketed leaderboard shall scope the results to the player's specific bucket.
- Bucketed leaderboards are used to foster high competitive engagement by pairing players with active, similarly-timed or similarly-skilled peers in small brackets.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Score submission ≤ 50ms (p99) |
| **Ranking** | Rank calculation ≤ 100ms for top-1000 queries |
| **Scalability** | Support 1,000+ concurrent leaderboards, millions of records each |
| **Consistency** | Scores shall be immediately reflected after write |
| **Availability** | 99.9% uptime |
| **Storage** | Efficient storage for large leaderboards |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Player identity for score ownership |
| BRD-06: Tournaments | Tournaments use leaderboards internally |
| BRD-18: Server Runtime & Hooks | Leaderboard event hooks |
| BRD-04: Authoritative Game Server | Server-side score submission |
| PostgreSQL | Record storage and ranking queries |
---
## 7. Acceptance Criteria

- [ ] Leaderboards can be created with configurable sort, operator, and reset schedule
- [ ] Scores are submitted and processed correctly (best/set/incr)
- [ ] Rankings are accurate and updated immediately
- [ ] Paginated listing returns correct records with ranks
- [ ] Around-player lookup returns a centered window
- [ ] Automated resets execute on schedule and optionally archive
- [ ] Authoritative leaderboards reject client-submitted scores
- [ ] Hooks fire on write and reset events
- [ ] Performance: score write ≤ 50ms, top-1000 query ≤ 100ms
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Ranking query performance at scale | Slow responses | Medium | Indexed queries; caching top-N; materialized ranks |
| Score manipulation (non-authoritative) | Unfair rankings | High | Authoritative mode; server-side validation hooks |
| Reset timing issues (timezone) | Inconsistent resets | Low | UTC-based cron; configurable timezone offset |
| Leaderboard spam | Storage bloat | Low | Rate limiting on writes; record TTL |
---
## 9. Future Considerations

- Friend-only leaderboards (scoped to social graph)
- Guild/clan leaderboards (aggregate scores)
- Historical leaderboard snapshots (view past seasons)
- Real-time rank change notifications via WebSocket
- Percentile-based rankings
- Anti-cheat score validation pipeline
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-05](../PRD/05_leaderboards.md) (API Surface & Interface Specification)
- [TDD-05](../TDD/05_leaderboards.md) (Database Schema & Technical Implementation Design)
