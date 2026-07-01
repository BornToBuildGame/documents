# BRD-15: Match State & Persistence

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Match State & Persistence  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Should Have
---
## 1. Purpose

Define the requirements for persisting match state, match history, replay metadata, and match statistics. This system enables game state saving/loading, match replay, post-match analysis, and historical statistics tracking.
---
## 2. Scope

### In Scope
- Match state snapshots (save/load)
- Match history records
- Replay metadata storage
- Post-match statistics
- Match result persistence
- Match search and querying

### Out of Scope
- Full game replay recording (frame-by-frame replay data — client concern)
- Video recording of matches
- Live match streaming
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Save/load game state, query match history |
| Players | View match history, statistics |
| Game Designers | Analyze match data, balance games |
| Data Analysts | Match analytics, player behavior |
---
## 4. Functional Requirements

### FR-MS-001: Match State Snapshot
- **Priority:** Must
- Server-side code shall save match state snapshots during or after a match.
- Snapshot fields:
  - `match_id`: UUID
  - `state`: JSON/binary (current game state)
  - `tick`: current tick number
  - `players`: list of participant IDs
  - `metadata`: JSON (match type, mode, map, etc.)
  - `create_time`: timestamp
- Multiple snapshots per match (checkpoint saves).
- Stored in PostgreSQL (or configurable storage backend).

### FR-MS-002: Match State Load
- **Priority:** Must
- Server-side code shall load a previously saved match state.
- Use cases:
  - Resume interrupted match
  - Load save game (single-player/co-op)
  - Post-match analysis
- Load by match_id and optional snapshot index/timestamp.

### FR-MS-003: Match History
- **Priority:** Must
- When a match ends, a match history record shall be created.
- Record fields:
  - `match_id`: UUID
  - `match_type`: string (game mode)
  - `players`: JSON array (user IDs, scores, stats)
  - `winner`: user_id or team_id (nullable for draws)
  - `duration`: int (seconds)
  - `start_time`: timestamp
  - `end_time`: timestamp
  - `metadata`: JSON (map, mode, settings)
  - `result`: JSON (final state summary)
- Match history is queryable by player.

### FR-MS-004: Player Match History
- **Priority:** Must
- Retrieve a player's match history.
- Parameters:
  - `user_id`
  - `limit` (default: 20, max: 100)
  - `cursor` (pagination)
  - `match_type` filter (optional)
- Results: list of match records with the player's stats per match.

### FR-MS-005: Replay Metadata
- **Priority:** Should
- Store metadata needed to replay a match.
- Replay metadata:
  - `match_id`: reference
  - `input_log_url`: location of input recording (if stored)
  - `initial_state`: starting state for deterministic replay
  - `random_seed`: server RNG seed
  - `tick_count`: total ticks
  - `player_inputs`: summary of input events
- Actual replay data storage is optional (can be external blob storage).

### FR-MS-006: Match Statistics
- **Priority:** Should
- Per-player per-match statistics:
  - Kills, deaths, assists
  - Damage dealt/taken
  - Resources gathered
  - Score
  - Time played
  - Custom stats (game-specific)
- Aggregate statistics per player (career stats):
  - Total matches played
  - Win/loss ratio
  - Average score
  - Best performance
- Stored in storage engine or dedicated stats table.

### FR-MS-007: Match Search
- **Priority:** Could
- Search historical matches by:
  - Player ID (matches containing this player)
  - Match type / mode
  - Date range
  - Result (win/loss)
  - Metadata fields
- Pagination support.

### FR-MS-008: Match State Cleanup
- **Priority:** Should
- Configurable retention period for match state snapshots.
- Match history retained longer than state data.
- Automatic cleanup of expired snapshots.
- Manual cleanup via admin API.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | State save ≤ 100ms (p99) |
| **Performance** | History query ≤ 200ms (p99) |
| **Scalability** | Support millions of match history records |
| **Storage** | Max state snapshot size: 1 MB |
| **Retention** | Configurable per match type (default: 90 days) |
| **Availability** | 99.9% uptime |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-03: Realtime Multiplayer | Match lifecycle (start/end events) |
| BRD-04: Authoritative Game Server | State generation in match handlers |
| BRD-18: Server Runtime & Hooks | Match end hooks for history writing |
| BRD-01: User Authentication | Player identity |
| PostgreSQL | State and history persistence |
---
## 7. Acceptance Criteria

- [ ] Match state can be saved as snapshots during or after a match
- [ ] Saved state can be loaded to resume a match
- [ ] Match history records are created when matches end
- [ ] Player match history is queryable with pagination
- [ ] Per-player match statistics are stored and retrievable
- [ ] State snapshots respect size limits
- [ ] Automatic cleanup of expired snapshots works correctly
- [ ] Performance: state save ≤ 100ms, history query ≤ 200ms
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Large state snapshots | Storage/performance | Medium | Size limits; compression |
| High-frequency saves | Database load | Medium | Save rate limiting; batching |
| Match history growth | Storage bloat | Medium | Retention policies; archival |
| State serialization bugs | Data corruption | Low | Validation on save; checksums |
---
## 9. Future Considerations

- Full replay recording system (input log + state snapshots)
- Replay playback API
- Match analytics pipeline (export to data warehouse)
- Heatmaps and spatial analytics
- Machine learning on match data (balance, cheat detection)
- Match sharing (social media, in-game replay gallery)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-15](../PRD/15_match_state_persistence.md) (API Surface & Interface Specification)
- [TDD-15](../TDD/15_match_state_persistence.md) (Database Schema & Technical Implementation Design)
