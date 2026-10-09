# Domain — Simple Stock Flow

> **Single source:** `spec/data-model.md`. Citations such as §x.y and FK-n refer to that document.
> Anything that does not come from the model is marked as **Assumption**.

## 1. Domain overview

The domain has **five entities** and **three aggregates** (§2). It handles a product catalog, immutable sales made of lines, and the internal users who register those sales. There is no customer entity (§1).

| Aggregate | Root | Internal parts | Source |
|---|---|---|---|
| Catalog | `Product` | Value objects `Money` and `Quantity` live in its row | §2.2 |
| Sales | `Sale` | `SaleItem` (no existence outside its sale) | §2.3, §2.4 |
| Identity | `User` | None | §2.5 |
| *(Reference, not an aggregate)* | `Category` | None; read-only, no lifecycle | §2.1 |

## 2. Entities and their rules

Each rule carries the model's mark: **engine** (enforced by Postgres), **domain-only** (enforced by C# code only) or **pending** (task not done yet).

### 2.1 `Category`

| Rule | Enforced by | Mark |
|---|---|---|
| Name required, non-empty, stored trimmed | `Category.Rename` | domain-only (moves to engine in T-20) |
| Name unique | Unique index `IX_category_name` | engine |

Five fixed rows are seeded in the initial migration; nobody creates, renames or deletes categories (§2.1, §9.1). Uniqueness is case- and accent-sensitive on purpose (§4.1).

### 2.2 `Product`

| Rule | Enforced by | Mark |
|---|---|---|
| Name required, non-empty, trimmed | `Product.Rename` | domain-only (`NOT NULL` is in the engine) |
| `price > 0` | `Product.ChangePrice` | domain-only (T-20) |
| `stock >= 0` after any operation | `Product.Withdraw` / `Product.Restock` | engine (`ck_product_stock_non_negative`) |
| Withdrawing more than the available stock fails | `Product.Withdraw` | domain-only (process rule, not expressible as a `CHECK`) |
| Category required and existing | `Product.SetCategory` + `FK_product_category_category_id` | engine |
| Missing image is `NULL`, never an empty string | `Product.AttachImage` | domain-only |
| Never physically deleted: soft delete | Shadow property `deleted_at` + global filter | engine (since T-09, per §13 D-1) |

A product has name, price, stock, category and an optional image, **and nothing else** (DP-03, §1).

### 2.3 `Sale`

| Rule | Enforced by | Mark |
|---|---|---|
| Records who made it; required and non-empty | `Sale` constructor | domain-only (`NOT NULL` is in the engine) |
| At least one line to be confirmed | `Sale.EnsureConfirmable` | domain-only |
| A product cannot repeat within a sale | `Sale.AddItem` | domain-only and engine (unique `(sale_id, product_id)`) |
| Withdrawing stock and adding the line are one operation | `Sale.AddItem` calls `Product.Withdraw` | domain-only |
| Immutable once registered | No edit or delete operation exists | domain-only (by absence) |

Total and subtotal are **computed, not stored** (§1).

### 2.4 `SaleItem`

| Rule | Enforced by | Mark |
|---|---|---|
| Product required | Constructor + `NOT NULL` + `FK_sale_item_product_product_id` (`RESTRICT`) | engine |
| `quantity > 0` | `Quantity` constructor | domain-only (T-20) |
| Product name and price frozen at sale time | `Sale.AddItem` copies from `Product` | domain-only |
| Category name frozen at sale time | Constructor + `NOT NULL` on `category_name` | engine per §2.4 (T-11) |
| Does not exist outside its sale | `FK_sale_item_sale_sale_id ON DELETE CASCADE` + `sale_id NOT NULL` | engine |

Its constructor is `internal`: only `Sale.AddItem` can create a line (§2.4).

### 2.5 `User`

| Rule | Enforced by | Mark |
|---|---|---|
| Username required and unique | Constructor + `IX_user_username` | engine (uniqueness) |
| Username lowercase and trimmed | `User.NormalizeUsername` | domain-only (T-20) |
| Password hash required, non-empty | `User` constructor | domain-only (`NOT NULL` is in the engine) |
| `role` in (`admin`, `seller`) | `Roles.IsValid` | domain-only (T-20) |
| The domain never sees the plain-text password | A hash port produces the hash (D-09) | By design |
