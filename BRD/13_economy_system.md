# BRD-13: Economy System

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Economy System  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Should Have
---
## 1. Purpose

Define the requirements for a game economy infrastructure supporting virtual currencies, player wallets, inventory management, item definitions, shops, crafting, and XP/leveling. While the server provides the foundational APIs and data structures, specific economy rules are implemented via server-side game logic.
---
## 2. Scope

### In Scope
- Player wallet (virtual currencies)
- Wallet transactions (credit, debit, atomic operations)
- Inventory management via storage system
- Item catalog (server-defined items)
- Virtual item shops and purchases using wallet currencies
- Integration with In-App Purchase (IAP) validation (see BRD-21)
- Economy-related RPC endpoints
- Transaction logging and auditing (ledger)

### Out of Scope
- Direct credit card processing or billing gateway integration (Stripe/PayPal)
- Economy balancing and tuning (game design concern)
- Dynamic pricing algorithms
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Designers | Balance economy, define items and pricing |
| Players | Fair economy, clear transactions |
| Game Developers | Reliable wallet/inventory APIs |
| Monetization Team | Economy health, currency sinks/faucets |
---
## 4. Functional Requirements

### FR-EC-001: Player Wallet
- **Priority:** Must
- Each user shall have a wallet stored in their account.
- Wallet is a JSON object containing currency balances.
- Example: `{"coins": 5000, "gems": 120, "xp": 34500}`
- Multiple currency types supported (developer-defined).
- Wallet is stored in the `users.wallet` JSONB field.

### FR-EC-002: Wallet Update
- **Priority:** Must
- Server-side code shall update wallet balances atomically.
- Operations:
  - Credit: add currency (e.g., quest reward)
  - Debit: subtract currency (e.g., purchase)
  - Mixed: credit some currencies, debit others (e.g., exchange)
- Negative balance prevention: debits shall fail if insufficient balance.
- Returns the updated wallet after the operation.
- Wallet updates generate a ledger entry for auditing.

### FR-EC-003: Wallet Ledger
- **Priority:** Must
- All wallet changes shall be logged in a ledger.
- Ledger entry fields:
  - `id`: UUID
  - `user_id`: UUID
  - `changeset`: JSON (the delta applied)
  - `metadata`: JSON (reason, source, context)
  - `create_time`: timestamp
- Ledger is append-only and immutable.
- Queryable by user_id with pagination.

### FR-EC-004: Item Catalog
- **Priority:** Should
- Server shall maintain an item catalog using storage objects.
- Item definition fields:
  - `item_id`: unique identifier
  - `name`: display name
  - `description`: text
  - `category`: (weapon, armor, consumable, cosmetic, etc.)
  - `rarity`: (common, uncommon, rare, epic, legendary)
  - `properties`: JSON (damage, defense, effect, duration, etc.)
  - `price`: JSON (cost in various currencies)
  - `stackable`: boolean
  - `max_stack`: integer
  - `tradeable`: boolean
- Catalog stored as global storage objects (permission_write=0).

### FR-EC-005: Inventory Management
- **Priority:** Should
- Player inventory stored using the Storage Engine (BRD-12).
- Inventory structure: collection=`"inventory"`, per-user documents.
- Operations (implemented as RPCs):
  - Add item to inventory
  - Remove item from inventory
  - Update item (upgrade, enchant)
  - List inventory with filtering
  - Transfer item between players (if tradeable)
- Inventory size limits (configurable).

### FR-EC-006: Virtual Shop Purchases
- **Priority:** Must
- All virtual shop purchases shall be validated server-side.
- Purchase flow:
  1. Client calls RPC (e.g., `BuyItem(item_id, quantity)`)
  2. Server validates:
     - Item exists in catalog
     - Player has sufficient virtual currency in their wallet
     - Item prerequisites met (level, etc.)
  3. Server atomically:
     - Debits wallet
     - Adds item to inventory
     - Logs transaction
  4. Returns result to client
- Client cannot directly modify wallet or inventory.

### FR-EC-010: In-App Purchase (IAP) Integration
- **Priority:** Must
- The economy system shall integrate with the IAP validation system (see BRD-21) to credit wallets or grant inventory items upon successful validation of real-money purchases.
- When an IAP receipt is validated successfully, the server shall credit the corresponding user's wallet (e.g., granting "gems") or insert the purchased item into the user's inventory.

### FR-EC-007: Crafting System
- **Priority:** Could
- Server-side crafting logic via RPCs.
- Crafting recipe: list of input items/currencies → output item.
- Crafting validation:
  - All input items present in inventory
  - Any currency cost deducted
  - Output item added to inventory
  - Input items consumed (removed)
- Recipes stored as global storage objects.

### FR-EC-008: Shop System
- **Priority:** Could
- Server-defined shops with item listings.
- Shop features:
  - Item availability (time-limited, stock limits)
  - Per-player purchase limits
  - Rotating shop (items change on schedule)
- Shop data stored as global storage objects.
- Purchase via RPC with server-side validation.

### FR-EC-009: XP and Leveling
- **Priority:** Should
- XP shall be a wallet currency (`xp`).
- Level calculation is server-side (XP thresholds per level).
- Level-up events trigger server hooks:
  - Unlock items
  - Grant rewards
  - Send notifications
- Level displayed on user profile (metadata or dedicated field).

### FR-EC-010: Economy RPC Endpoints
- **Priority:** Must
- Developer-defined RPC functions for economy actions:

| RPC | Description |
|-----|-------------|
| `BuyItem(item_id, quantity)` | Purchase item from shop |
| `SellItem(item_id, quantity)` | Sell item for currency |
| `CraftItem(recipe_id)` | Craft an item |
| `ClaimDailyReward()` | Claim daily login bonus |
| `OpenChest(chest_id)` | Open loot chest |
| `UpgradeWeapon(item_id)` | Upgrade a weapon |
| `TradeItem(target_user, item_id)` | Trade item with player |
| `ExchangeCurrency(from, to, amount)` | Currency exchange |
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Atomicity** | Wallet + inventory operations shall be atomic |
| **Performance** | Wallet update ≤ 50ms (p99) |
| **Auditability** | All wallet changes logged in immutable ledger |
| **Security** | No client-side modification of wallet/inventory |
| **Scalability** | Support millions of wallets and inventories |
| **Consistency** | Strong consistency for financial operations |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Player identity for wallet ownership |
| BRD-12: Storage Engine | Inventory and item catalog storage |
| BRD-14: RPC & Custom APIs | Economy RPC endpoints |
| BRD-04: Authoritative Game Server | Server-side economy logic |
| BRD-18: Server Runtime & Hooks | Economy event processing |
| BRD-11: Notifications | Purchase/reward notifications |
| PostgreSQL | Wallet and ledger persistence |
---
## 7. Acceptance Criteria

- [ ] Player wallets support multiple currency types
- [ ] Wallet updates are atomic — no partial credit/debit
- [ ] Negative balance is prevented on debit operations
- [ ] All wallet changes are logged in an immutable ledger
- [ ] Item catalog can be defined as global storage objects
- [ ] Player inventory is stored via the storage engine with server-only write
- [ ] Purchase RPCs validate server-side (balance, item existence, prerequisites)
- [ ] Wallet + inventory changes happen atomically within a purchase
- [ ] Wallet ledger is queryable with pagination
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Currency duplication exploits | Economy collapse | High | Atomic operations; server-only writes; ledger audit |
| Race conditions on wallet | Double-spend | Medium | Database-level atomicity; row-level locks |
| Inventory overflow | Data issues | Low | Size limits; stack limits; validation |
| Economy inflation | Game balance issues | Medium | Sink/faucet monitoring; analytics (game design) |
| Ledger storage growth | Database bloat | Low | Ledger archival; retention policies |
---
## 9. Future Considerations

- Real-money purchase validation (IAP receipt verification)
- Trading marketplace (player-to-player economy)
- Auction house
- Loot box / gacha system with configurable drop rates
- Currency exchange rates
- Economy analytics dashboard
- Gifting system
- Subscription / battle pass support
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-13](../PRD/13_economy_system.md) (API Surface & Interface Specification)
- [TDD-13](../TDD/13_economy_system.md) (Database Schema & Technical Implementation Design)
