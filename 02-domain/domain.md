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


## 3. Value objects

| Value object | Rule | Source |
|---|---|---|
| `Money` | Rounds to 2 decimals with `MidpointRounding.AwayFromZero`; rejects negatives but **accepts zero**; matches the `numeric(18,2)` column | §2.2 |
| `Quantity` | Strictly positive | §1, §2.4 |
| Date range | The end cannot precede the start; application-layer object, no table | §1 |

Note: `price > 0` is guarded only by `Product.ChangePrice`, because `Money` allows zero (§2.2).

## 4. Relationships

| From | To | Cardinality | Nature | Source |
|---|---|---|---|---|
| `category` | `product` | 1:N | Cross-aggregate, by root identity (FK-1, `RESTRICT`) | §5 |
| `sale` | `sale_item` | 1:N | Internal composition (FK-2, `CASCADE`) | §5 |
| `sale_item` | `product` | N:1 | Cross-aggregate, by root identity (FK-3, `RESTRICT`) | §5 |
| `sale` | `user` | N:1 | Cross-aggregate, by identity (FK-4, `RESTRICT`, **pending T-12**) | §5 |

The only N:M relationship is `sale` ↔ `product`, resolved by `sale_item`, which carries its own data (`quantity`, `unit_price`, `product_name`, `category_name`) (§5).

## 5. Domain events

**Assumption:** the model does not list events. The following are derived from its operations and are not stated in it.

| Event (assumed) | Triggered by | Derived from |
|---|---|---|
| `ProductCreated` | Creating a product | `Product.Rename`, `SetCategory` (§2.2) |
| `ProductPriceChanged` | Changing a product's price | `Product.ChangePrice` (§2.2) |
| `ProductDeactivated` | Setting `deleted_at` | Soft delete (§2.2, §7.1) |
| `StockWithdrawn` | Adding a line to a sale | `Product.Withdraw` (§2.2, §2.3) |
| `StockReplenished` | Adding stock | `Product.Restock` (§2.2) |
| `SaleRegistered` | Confirming a sale | `Sale.EnsureConfirmable` (§2.3) |


## 6. Glossary

| Business term | Functional definition | Technical home |
|---|---|---|
| Product (*Producto*) | Catalog item: name, price, stock, category and optional image | `Product` · table `product` |
| Category (*Categoría*) | One of five fixed classifications | `Category` · table `category` |
| Price (*Precio*) | Current monetary value of the product; strictly positive | `Money` · `product.price` |
| Stock | Units available; never negative | `product.stock` |
| Product image (*Imagen*) | Opaque key to an external binary; `NULL` when absent | `product.image_key` |
| Sale (*Venta*) | Consummated, immutable commercial fact: who, when and what | `Sale` · table `sale` |
| Sale line (*Línea de venta*) | Sale row: product, quantity and frozen price | `SaleItem` · table `sale_item` |
| Quantity (*Cantidad*) | Units sold in a line; strictly positive | `Quantity` · `sale_item.quantity` |
| Sale total (*Total*) | Sum of subtotals; computed, not stored | `Sale.Total` · no column |
| Line subtotal (*Subtotal*) | Unit price times quantity; computed, not stored | `SaleItem.Subtotal` · no column |
| User (*Usuario*) | Internal operator who logs in and registers sales; there is no customer | `User` · table `user` |
| Role (*Rol*) | `admin` or `seller` | `user.role` |
| Password hash (*Hash de clave*) | Irreversible fingerprint of the password | `user.password_hash` |
| Date range (*Rango de fechas*) | Time window for the report | Application-layer value object |
| Sales report (*Reporte de ventas*) | Per-product aggregation over a range; not persisted | Read model |
| Frozen (*Congelado*) | A copy of a value taken at sale time that does not follow the catalog | `sale_item` columns |

## 7. Closed decisions that bound the domain

- Single currency, no currency column (§3, D-05).
- No audit columns (§8).
- No per-seller report breakdown (DP-02, §7.1).
- No extra product attributes (DP-03, §1).
- Nobody grants `admin` at runtime (DP-04, §11 H-3).

## 8. Open points

- **Possible conflict:** `spec.md` CA-06.1 says "one row per product", but §11.1 yields more than one row when a product was recategorized.
- **Inconsistency noticed in the model:** §3 lists `category_name` among `product` columns, but it belongs to `sale_item`.
- **Column count:** §3 says 22 columns, but the table lists 21 plus `xmin`, which is a system column (§3, §10.1).
