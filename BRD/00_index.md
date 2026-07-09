# Game Server — Business Requirements Documents (BRD) Index

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Version:** 1.2  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Reference Architecture:** [Ultimate Game Engine](https://heroiclabs.com/nakama/) — The leading open-source game backend by Heroic Labs

---

## Overview

This directory contains the Business Requirements Documents (BRD) for the **Ultimate Game Engine Multiplayer Game Server**. The server follows [Ultimate Game Engine](https://heroiclabs.com/nakama/), the leading open-source game backend framework for building online multiplayer games. It provides a complete backend infrastructure enabling developers to focus on gameplay rather than backend engineering.

The server is designed for use with the following game engines and frameworks:

| Engine / Framework | SDK Language |
|-------------------|-------------|
| **Godot** (3 & 4) | GDScript |
| **Unity** | C# / .NET |
| **Unreal Engine** | C++ |
| **MonoGame** | C# / .NET |
| **LibGDX** | Java |
| **Defold** | Lua |
| **Cocos2d** (C++ & JS) | C++ / JavaScript |
| **Phaser** | JavaScript |
| **Macroquad** | Rust |
| **Custom Engines** | C++, JavaScript, Swift, Dart/Flutter, Java/Android |

---

## Architecture Overview

```
             Game Clients (Godot, Unity, Unreal, Defold, Phaser, etc.)
                      │
         ┌────────────┼────────────┐
         │            │            │
     REST API     WebSocket     gRPC
     (HTTP/1.1)   (Realtime)   (HTTP/2)
         │            │            │
         └────────────┼────────────┘
                      │
               Game Server Core
       ┌──────────────┼──────────────┐
       │              │              │
  Authentication   Matchmaker     Chat
       │              │              │
  Leaderboards     Parties       Guilds
       │              │              │
  Storage         RPC Logic    Notifications
       │              │              │
  IAP Validation   Presence    Economy/Wallet
       └──────────────┼──────────────┘
                      │
      ┌───────────────┼───────────────┐
      │               │               │
  PostgreSQL    Runtime (Go/TS/Lua)  Metrics
  (Primary DB)  (Server Logic)     (Prometheus)
```

---

## Document Catalog

This master index maps all modules to their corresponding requirements and specifications.

| # | Feature / Module | Feature Area | BRD (Business Rules) | PRD (API Specs) | TDD (DB Schemas) |
|---|------------------|-------------|----------------------|-----------------|------------------|
| 01 | User Authentication | Identity & Access | [BRD-01](./01_user_authentication.md) | [PRD-01](../PRD/01_user_authentication.md) | [TDD-01](../TDD/01_user_authentication.md) |
| 02 | Multiplayer Matchmaking | Multiplayer | [BRD-02](./02_multiplayer_matchmaking.md) | [PRD-02](../PRD/02_multiplayer_matchmaking.md) | [TDD-02](../TDD/02_multiplayer_matchmaking.md) |
| 03 | Realtime Multiplayer | Multiplayer | [BRD-03](./03_realtime_multiplayer.md) | [PRD-03](../PRD/03_realtime_multiplayer.md) | [TDD-03](../TDD/03_realtime_multiplayer.md) |
| 04 | Authoritative Game Server | Game Logic | [BRD-04](./04_authoritative_game_server.md) | [PRD-04](../PRD/04_authoritative_game_server.md) | [TDD-04](../TDD/04_authoritative_game_server.md) |
| 05 | Leaderboards | Competitive | [BRD-05](./05_leaderboards.md) | [PRD-05](../PRD/05_leaderboards.md) | [TDD-05](../TDD/05_leaderboards.md) |
| 06 | Tournaments | Competitive | [BRD-06](./06_tournaments.md) | [PRD-06](../PRD/06_tournaments.md) | [TDD-06](../TDD/06_tournaments.md) |
| 07 | Friends System | Social | [BRD-07](./07_friends_system.md) | [PRD-07](../PRD/07_friends_system.md) | [TDD-07](../TDD/07_friends_system.md) |
| 08 | Parties | Social | [BRD-08](./08_parties.md) | [PRD-08](../PRD/08_parties.md) | [TDD-08](../TDD/08_parties.md) |
| 09 | Guilds & Clans | Social | [BRD-09](./09_guilds_clans.md) | [PRD-09](../PRD/09_guilds_clans.md) | [TDD-09](../TDD/09_guilds_clans.md) |
| 10 | Chat System | Communication | [BRD-10](./10_chat_system.md) | [PRD-10](../PRD/10_chat_system.md) | [TDD-10](../TDD/10_chat_system.md) |
| 11 | Notifications | Communication | [BRD-11](./11_notifications.md) | [PRD-11](../PRD/11_notifications.md) | [TDD-11](../TDD/11_notifications.md) |
| 12 | Storage Engine | Data | [BRD-12](./12_storage_engine.md) | [PRD-12](../PRD/12_storage_engine.md) | [TDD-12](../TDD/12_storage_engine.md) |
| 13 | Economy System | Game Economy | [BRD-13](./13_economy_system.md) | [PRD-13](../PRD/13_economy_system.md) | [TDD-13](../TDD/13_economy_system.md) |
| 14 | RPC & Custom APIs | Extensibility | [BRD-14](./14_rpc_custom_apis.md) | [PRD-14](../PRD/14_rpc_custom_apis.md) | [TDD-14](../TDD/14_rpc_custom_apis.md) |
| 15 | Match State & Persistence | Data | [BRD-15](./15_match_state_persistence.md) | [PRD-15](../PRD/15_match_state_persistence.md) | [TDD-15](../TDD/15_match_state_persistence.md) |
| 16 | Groups | Social | [BRD-16](./16_groups.md) | [PRD-16](../PRD/16_groups.md) | [TDD-16](../TDD/16_groups.md) |
| 17 | Presence System | Realtime | [BRD-17](./17_presence_system.md) | [PRD-17](../PRD/17_presence_system.md) | [TDD-17](../TDD/17_presence_system.md) |
| 18 | Server Runtime & Hooks | Extensibility | [BRD-18](./18_server_runtime_hooks.md) | [PRD-18](../PRD/18_server_runtime_hooks.md) | [TDD-18](../TDD/18_server_runtime_hooks.md) |
| 19 | API Layer (REST, gRPC, WebSocket) | Infrastructure | [BRD-19](./19_api_layer.md) | [PRD-19](../PRD/19_api_layer.md) | [TDD-19](../TDD/19_api_layer.md) |
| 20 | Database & Infrastructure | Infrastructure | [BRD-20](./20_database_infrastructure.md) | [PRD-20](../PRD/20_database_infrastructure.md) | [TDD-20](../TDD/20_database_infrastructure.md) |
| 21 | In-App Purchase Validation | Monetization | [BRD-21](./21_iap_validation.md) | [PRD-21](../PRD/21_iap_validation.md) | [TDD-21](../TDD/21_iap_validation.md) |
| 22 | Console Admin | Developer Tools | [BRD-22](./22_console_admin.md) | [PRD-22](../PRD/22_console_admin.md) | [TDD-22](../TDD/22_console_admin.md) |

---

## Feature Areas

### Identity & Access
- User Authentication (multi-provider, session management, account linking)

### Multiplayer
- Matchmaking (automatic, skill-based, party, offline matchmaking, count multiple)
- Realtime Multiplayer (client-relayed, authoritative, session-based)
- Authoritative Game Server (server-side game logic, match handlers)

### Competitive
- Leaderboards (ranked, seasonal, paginated, bucketed)
- Tournaments (scheduled, seasonal, rewards, tiered leagues)

### Social
- Friends System (social graph, presence, blocking)
- Parties (co-op grouping, party matchmaking, party chat)
- Guilds & Clans (persistent organizations, roles, permissions)
- Groups (communities, clubs, teams)

### Communication
- Chat System (DM, group, guild, match, party)
- Notifications (in-app, persistent, server-driven)

### Data
- Storage Engine (NoSQL documents, permissions, search, collections)
- Match State & Persistence (game state, replays, snapshots)

### Game Economy
- Economy System (currencies, inventory, wallet)

### Monetization
- In-App Purchase Validation (Apple, Google, Huawei, Facebook Instant)

### Extensibility
- RPC & Custom APIs (custom backend functions)
- Server Runtime & Hooks (Go, TypeScript, Lua runtimes; before/after hooks; events; cron)

### Realtime
- Presence System (online status, streams, subscriptions)

### Infrastructure
- API Layer (REST, gRPC, WebSocket; client SDKs)
- Database & Infrastructure (PostgreSQL, scaling, monitoring, admin console)

---

## Ultimate Game Engine Feature Parity

This server follows Ultimate Game Engine's feature set. The following Ultimate Game Engine concepts are mapped to our BRDs:

| Ultimate Game Engine Concept | BRD |
|---------------|-----|
| Authentication | BRD-01 |
| Sessions / User Accounts | BRD-01 |
| Storage Engine (Collections, Permissions, Search) | BRD-12 |
| Friends | BRD-07 |
| Groups and Clans | BRD-09, BRD-16 |
| Real-time Chat | BRD-10 |
| Parties | BRD-08 |
| Multiplayer Engine (Relayed, Authoritative, Session-Based) | BRD-03, BRD-04 |
| Matchmaker (inc. Offline Matchmaking) | BRD-02 |
| Match Listing | BRD-03 |
| Status / Presence | BRD-17 |
| Leaderboards (inc. Bucketed) | BRD-05 |
| Tournaments | BRD-06 |
| Notifications | BRD-11 |
| In-App Purchase Validation | BRD-21 |
| Events | BRD-18 (FR-SR-005) |
| Sockets | BRD-19 |
| Server Framework (Go, TypeScript, Lua) | BRD-18 |
| Hooks (Before/After) | BRD-18 |
| Streams | BRD-17 |
| RPC | BRD-14 |
| Client Libraries | BRD-19 (FR-API-013) |
| Console (Admin) | BRD-20 (FR-DB-009) |
| Metrics | BRD-20 (FR-DB-007) |

---

## Conventions

- Each **BRD** focuses strictly on high-level business motivation, MoSCoW prioritization, functional scope, and acceptance criteria.
- API endpoints and serialization protocol payloads are isolated in **PRDs**.
- PostgreSQL table definitions, query indexes, and code blocks are isolated in **TDDs**.
- Feature naming follows Ultimate Game Engine conventions where applicable for developer familiarity.

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD creation for all 20 feature areas |
| 1.1 | 2026-07-01 | Engineering | Added Ultimate Game Engine reference; added BRD-21 (IAP Validation) |
| 1.2 | 2026-07-01 | Engineering | Cleaned BRDs of technical specs; split API specs into PRDs and DB schemas into TDDs |
