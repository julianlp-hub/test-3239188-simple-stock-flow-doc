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


## 4. Constraints from the model

- The schema is owned by EF migrations and nothing else (§3.2).
- If the document contradicts the engine, **the engine wins** and the document is broken (header, "Rige bajo").
- All timestamps are `timestamptz` and the server runs in UTC (§3).
- Sales and lines are kept indefinitely and never edited or deleted (§7.1).
- The password hash is never logged, returned or indexed (§7).

## 5. External dependencies

| Dependency | Use | Source |
|---|---|---|
| PostgreSQL 16 | Stores the five tables and enforces `engine` rules | §3, §10 |
| External image storage | Holds image binaries; the model stores only an opaque key | §1, §7.1 |
| Password hashing port | Produces `password_hash`; the domain never sees the plain password | §2.5, §9.2 |
| Application startup | Creates the initial administrator from environment credentials | §9.2 |

## 6. Assumptions

- **Assumption:** the system is used in a small retail business, suggested by the five seeded categories (§9.1) and by the absence of customers and payments (§1, §7).
- **Assumption:** the system is exposed through an API, which the model mentions but does not describe (§12).
- **Assumption:** the document dates (2026-09-19 for the base, 2026-09-20 for the debt register) mean that §13 reflects a state more recent than the query outputs in §10.

## 7. Open items and inconsistencies found

- **Pending work:** `sale.sold_by_user_id` and FK-4 (T-12), search and report indexes (T-13), and the `CHECK` constraints for `price > 0`, `quantity > 0`, role and non-empty name (T-20) (§4, §5, §6.2).
- **Open decision:** `spec.md` CA-06.1 says "one row per product", but §11.1 yields more than one when a product was recategorized.
- **Inconsistency:** §3 says 22 columns while the table lists 21 plus the system column `xmin` (§3, §10.1).
- **Inconsistency:** §3 lists `category_name` among `product` columns, but it belongs to `sale_item`.
- **Stale snapshot:** §10 shows the schema as of 2026-09-19; §13 records that `deleted_at`, `sale_id NOT NULL`, the unique `(sale_id, product_id)` index and FK-3 were settled on 2026-09-20.
