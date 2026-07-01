# PRD-16: Groups

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Groups  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a general-purpose group system, distinct from guilds/clans, for organizing players into communities, clubs, schools, teams, and other non-competitive associations. Groups share the same underlying data model and API surface as guilds but serve different use cases.

---

Refer to [BRD-16](../BRD/16_groups.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Groups share the HTTP/REST and gRPC API surface with Guilds & Clans (see [PRD-09](./09_guilds_clans.md)). Distinct group categories (e.g. Community, Club, Team) are specified using integer category identifiers during creation and filtering.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

Groups share the GroupService and REST group endpoints defined in [PRD-09: Guilds & Clans](./09_guilds_clans.md#3-api-surface-detail). The specifications are extended here to document the category field values and group-specific usage.

### Group Category Codes
When creating or searching groups, the `category` field is used to filter by group type:

| Category Name | Value | Description |
|---------------|-------|-------------|
| Guild / Clan  | `0`   | Competitive guilds (documented in PRD-09) |
| Community     | `1`   | General social communities |
| Club          | `2`   | Interest-based player clubs |
| School / Class| `3`   | Educational / coaching groups |
| Team          | `4`   | Cooperative multiplayer squads/teams |
| Custom        | `5+`  | Developer-defined custom groups |

### REST Endpoint Extensions

#### 1. Create Group (Non-Guild)
* **Endpoint**: `POST /v2/group`
* **Request Body**:
  ```json
  {
    "name": "speedrunners_club",
    "description": "Speedrunning discussion and runs coordination.",
    "avatar_url": "https://example.com/speedrun.jpg",
    "lang_tag": "en-US",
    "open": true,
    "max_count": 500,
    "category": 2,
    "metadata": {
      "focus_game": "ultimate_racer"
    }
  }
  ```
* **Response Body (200 OK)**:
  ```json
  {
    "id": "group-uuid-3333-4444",
    "creator_id": "e932b70f-152e-436d-9614-22b274be59c0",
    "name": "speedrunners_club",
    "description": "Speedrunning discussion and runs coordination.",
    "avatar_url": "https://example.com/speedrun.jpg",
    "lang_tag": "en-US",
    "open": true,
    "edge_count": 1,
    "max_count": 500,
    "category": 2,
    "metadata": "{\"focus_game\":\"ultimate_racer\"}",
    "create_time": "2026-07-01T22:30:00Z",
    "update_time": "2026-07-01T22:30:00Z"
  }
  ```

#### 2. Search Groups by Category
* **Endpoint**: `GET /v2/group`
* **Query Parameters**:
  * `category` (integer, required): Filter by specific group type (e.g. `2` for Clubs).
  * `name` (string, optional): Search query prefix.
  * `limit` (integer, optional, default: `20`): Max results.
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**: Returns the matching list of group objects (same format as PRD-09).

---

### gRPC Service Protocol
Groups utilize the shared `GroupService` interface. Refer to [PRD-09: Guilds & Clans](./09_guilds_clans.md#grpc-service-protocol) for the complete `GroupService` protobuf definitions.

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `group.max_non_guild_joined` | integer | `10` | Maximum number of non-guild groups a player can join. |
| `group.default_max_members` | integer | `500` | Default member limit applied to new general groups. |

---

## Linked Documents
- [BRD-16](../BRD/16_groups.md) (Business Requirements Document)
- [TDD-16](../TDD/16_groups.md) (Technical Design Document)
