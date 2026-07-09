# ADR-0007: Leaderboard, Tournament, and Ephemeral Party Alignment

## Status
Accepted

## Date
2026-07-09

## Context
A gaps-and-mismatches review of the Game Server's specifications and current implementation was performed against the reference architecture's core capabilities:
1. **Leaderboards & Tournaments (TDD-05, TDD-06)**: Although detailed in the TDD files, their database migrations, Go package logic, and API gateway binds were completely omitted from the codebase and master implementation checklist.
2. **Ephemeral Parties (TDD-08)**: Ephemeral lobby and transient real-time group capabilities were designed in TDD, but the in-memory maps, session lifecycles, and socket packet dispatches (`party_create`, `party_invite`, `party_join`) are not yet implemented.
3. **Multi-Node scaling**: The architectural specs call for distributed state scaling (Redis Pub/Sub relaying for sockets and peer-to-peer gRPC meshes for authoritative loops), but the active implementation is single-node only.

To achieve parity with the target specifications, these modules must be officially added to the Software Architecture Document (SAD), High-Level Design (HLD), and the master Implementation Plan.

## Decision
Align the architecture and implementation checklist to fully implement Leaderboards, Tournaments, and Ephemeral Parties as described in TDD-05, TDD-06, and TDD-08:

1. **Database Schema Integration**: Incorporate the `leaderboard` and `leaderboard_record` tables, constraints, and index strategy into the migrations pipeline (Phase 11).
2. **Leaderboard & Tournament Engine**: Implement an in-memory rank caching system synchronized across cluster nodes using Redis Pub/Sub invalidations. Implement a cron-based scheduler to tick active occurrences and evaluate tournament endings.
3. **Ephemeral Party Gateway**: Implement a thread-safe in-memory party registry. Support socket commands for party control (Create, Join, Leave, Invite) with automated leader promotion.
4. **Implementation Plan Realignment**: Expand the implementation checklist with Phase 11 (Leaderboards & Tournaments) and Phase 12 (Ephemeral Parties) to track progress.

## Consequences

### Positive
- **100% Specification Parity**: Brings the server database and engine to parity with the design documents and the core feature set.
- **Robust Scalability**: Ensures that real-time socket updates and authoritative loops scale cleanly using Redis and peer-to-peer gRPC routing.
- **Clear Roadmap**: Extends the master checklist with clear verification test suites for each missing component.

### Negative
- **Increased Memory Profile**: Ephemeral party state maps and leaderboard dynamic rank caches will consume additional memory on server nodes.
- **Complexity**: Background schedulers and synchronization pub/sub channels require careful lifecycle management to prevent goroutine leaks.
