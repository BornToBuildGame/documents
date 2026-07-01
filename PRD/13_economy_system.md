# PRD-13: Economy System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Product Requirements:** Economy System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Product Specification

---

## 1. Purpose & Scope

Define the requirements for a game economy infrastructure supporting virtual currencies, player wallets, inventory management, item definitions, shops, crafting, and XP/leveling. While the server provides the foundational APIs and data structures, specific economy rules are implemented via server-side game logic.

---

Refer to [BRD-13](../BRD/13_economy_system.md) for the complete business requirements.

---

## 2. Protocol & Transport Specifications

Wallet and ledger retrieval operations are available over REST and gRPC. To prevent cheating, balance mutations (credits/debits) can only be triggered via server runtime scripts executing on the authoritative server.

### REST Headers
* **Content-Type**: `application/json`
* **Authorization**: `Bearer <session_token>`

---

## 3. API Surface Detail

### REST Endpoints

#### 1. Get Wallet
Retrieves the user's current virtual currency balances.
* **Endpoint**: `GET /v2/wallet`
* **Response Body (200 OK)**:
  ```json
  {
    "wallet": {
      "coins": 5000,
      "gems": 120,
      "xp": 34500
    }
  }
  ```

#### 2. Get Wallet Ledger
Retrieves an audit history of all wallet modifications.
* **Endpoint**: `GET /v2/wallet/ledger`
* **Query Parameters**:
  * `limit` (integer, optional, default: `20`, max: `100`): Max entries page size.
  * `cursor` (string, optional): Pagination token.
* **Response Body (200 OK)**:
  ```json
  {
    "items": [
      {
        "id": "ledger-uuid-3333-4444",
        "user_id": "e932b70f-152e-436d-9614-22b274be59c0",
        "changeset": {
          "coins": 500
        },
        "metadata": {
          "source": "quest_reward",
          "quest_id": "daily_quest_1"
        },
        "create_time": "2026-07-01T22:30:00Z"
      }
    ],
    "next_cursor": "page_token_string"
  }
  ```

---

### gRPC Service Protocol

```protobuf
syntax = "proto3";

package ultimate.server.api;

import "google/protobuf/timestamp.proto";

service EconomyService {
  rpc GetWallet(GetWalletRequest) returns (WalletResponse);
  rpc ListWalletLedger(ListWalletLedgerRequest) returns (WalletLedgerList);
}

message GetWalletRequest {
  string user_id = 1;
}

message WalletResponse {
  string wallet = 1; // JSON string format containing balances (e.g. {"coins": 500})
}

message ListWalletLedgerRequest {
  string user_id = 1;
  int32 limit = 2;
  string cursor = 3;
}

message WalletLedgerItem {
  string id = 1;
  string user_id = 2;
  string changeset = 3; // JSON string representation
  string metadata = 4;  // JSON string representation
  google.protobuf.Timestamp create_time = 5;
}

message WalletLedgerList {
  repeated WalletLedgerItem items = 1;
  string next_cursor = 2;
}
```

---

### Server Runtime API

These methods are available for scripting wallet adjustments inside match loops, webhooks, or RPC handlers:

```lua
-- Delta changes: positive is credit, negative is debit
local changeset = { coins = 100, gems = -10 }
local metadata = { source = "shop_purchase", item_id = "sword_01" }

local updated_wallet, updated_ledger = nk.wallet_update(user_id, changeset, metadata, true)
```

#### Multi-wallet updates (Atomic batch):
```lua
local updates = {
  { user_id = "user-1", changeset = { coins = 10 }, metadata = { source = "event" } },
  { user_id = "user-2", changeset = { coins = 10 }, metadata = { source = "event" } }
}
nk.wallets_update(updates, true)
```

---

## 4. Product Configurations

| Parameter Name | Type | Default Value | Description |
|----------------|------|---------------|-------------|
| `economy.allow_negative_balances` | boolean | `false` | If false, transaction fails if wallet balances fall below zero. |
| `economy.ledger_retention_days` | integer | `90` | Number of days wallet change logs are kept before being archived. |

---

## Linked Documents
- [BRD-13](../BRD/13_economy_system.md) (Business Requirements Document)
- [TDD-13](../TDD/13_economy_system.md) (Technical Design Document)
