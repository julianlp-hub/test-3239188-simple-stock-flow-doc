# Context — Simple Stock Flow

> **Single source:** `spec/data-model.md`. Citations such as §x.y refer to that document.
> Anything that does not come from the model is marked as **Assumption**.

## 1. General description

Simple Stock Flow is an inventory and sales system. It keeps a catalog of products with stock, records sales made of lines, and produces a per-product sales report over a date range (§1, §2, §6.1).

It is used only by internal operators with two roles, `admin` and `seller`. There is no customer or buyer entity (§1, §2.5).

The model is stored in PostgreSQL 16, in the `sales` schema, with five tables: `category`, `product`, `sale`, `sale_item` and `user` (§0, §3).

## 2. Scope

### 2.1 In scope

| Capability | Source |
|---|---|
| Product catalog: name, price, stock, category, optional image | §1, §2.2 |
| Five fixed, seeded, read-only categories | §2.1, §9.1 |
| Immutable sales with at least one line, with frozen name, price and category | §2.3, §2.4 |
| Stock withdrawal in the same operation as the sale line | §2.3 |
| Internal users with a role and a password hash | §2.5 |
| Soft delete of products | §2.2, §7.1 |
| Per-product sales report over a date range, computed in the engine | §1, §6.1 (Q9) |
| Product search by text and category | §6.1 (Q1) |

### 2.2 Out of scope

| Excluded | Why | Source |
|---|---|---|
| Customers and buyers | No customer entity; only the internal operator is recorded | §1, §7 |
| Payments and cards | No payment data | §7 |
| Multi-currency | Single-currency system, no currency column | §3 (D-05) |
| Audit columns (`created_at`, `updated_at`) | No requirement; decision closed | §8 |
| Category maintenance (create, rename, delete) | Categories are seed data | §2.1, §4.1 |
| Extra product attributes (description, SKU, reference code) | Decision DP-03 | §1, §12 |
| Per-seller report breakdown | Decision DP-02 | §7.1, §12 |
| The API contract | Lives in a separate document, not delivered | §12 |
| Business requirements and acceptance criteria | Live in `spec.md`, not delivered | §12 |

## 3. Stakeholders and users

| Role | Interest | Source |
|---|---|---|
| Seller (`seller`) | Registers sales, looks up products | §2.5 |
| Administrator (`admin`) | Creates sellers, maintains the catalog, reads reports | §2.5, §11 (H-3) |
| Model owner | Decides open points such as H-2 and the CA-06.1 wording | §11, §11.1 |
