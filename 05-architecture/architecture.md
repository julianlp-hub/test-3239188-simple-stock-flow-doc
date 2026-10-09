# Architecture — Simple Stock Flow

> **Single source:** `spec/data-model.md`. Citations such as §x.y refer to that document.
> Anything that does not come from the model is marked as **Assumption**.

## 1. Architectural style

**Hexagonal architecture (ports and adapters).** The domain knows nothing about the database or external services; it talks to them through ports.

Evidence in the model: the "persistence adapter" translates between classes and tables (§0), the "hash port" produces the password hash (§2.5, §9.2), and a "read port" computes the report (§1, §6.1).

## 2. System components

| Component | Type | Responsibility | Source |
|---|---|---|---|
| `Category` | Reference entity | Five fixed categories, read-only, no lifecycle | §2.1, §9.1 |
| `Product` | Aggregate root (catalog) | Name, price, stock, category, image; soft delete | §2.2 |
| `Sale` | Aggregate root (sales) | Immutable sale: who, when and what | §2.3 |
| `SaleItem` | Entity internal to `Sale` | Line with product, quantity and frozen price | §2.4 |
| `User` | Aggregate root (identity) | User, role (`admin` / `seller`) and password hash | §2.5 |
| `Money`, `Quantity` | Value objects | No identity or table; they live in their owner's row | §2 |
| Hash port | Outbound port | The domain never sees the plain-text password | §2.5, §9.2 |
| Report read port | Outbound port | Per-product aggregation computed in the engine | §1, §6.1 (Q9) |
| Persistence adapter | Adapter | Maps classes to tables; C# collections are plural, tables singular | §0 |
| Image storage | External system | Stores the binary; the model keeps only an opaque key | §1, §7.1 |
| Application startup | Process | Creates the initial administrator from environment credentials | §9.2 |

## 3. Where each rule lives

The model classifies every rule with one of three marks (section "How to read this document", §4):

- **engine** (guaranteed by Postgres): `stock >= 0` (`ck_product_stock_non_negative`), uniqueness of `category.name` and `user.username`, foreign keys FK-1, FK-2 and FK-3, unique `(sale_id, product_id)`.
- **domain-only** (guaranteed by C# code): `price > 0`, `quantity > 0`, valid role, lowercase username, a sale with at least one line, withdrawing more stock than available.
- **pending**: `sale.sold_by_user_id` and FK-4 (T-12), search indexes (T-13), `CHECK` constraints to be moved into the engine (T-20).

**Implication:** a rule that lives only in the domain protects the application but not the data; a manual `INSERT` bypasses it (section "How to read this document").

## 4. Structural decisions implied by the model

| Decision | Reason | Source |
|---|---|---|
| The schema is owned by migrations | All DDL goes through a single path | §3.2 |
| Optimistic concurrency with `xmin` | Prevents overselling when stock is decremented | §3 (`xmin`), §2.2 |
| Soft delete of products (`deleted_at`) | Sale lines and the report depend on the row | §2.2, §7.1 |
| Name, price and category frozen in `SaleItem` | Repricing or renaming does not rewrite history | §1, §2.4 |
| Total and subtotal are computed, not stored | Avoids two sources of truth | §1 |
| Report is not persisted | Computed in the engine over a date range | §1, §6.2 |
| Report groups by the frozen category | A closed report must never change | §11.1 |
| Single-currency system, no currency columns | Closed decision | §3 |
| No audit columns | There is no requirement for them | §8 |

## 5. Assumptions

- **Assumption:** the system is exposed through an API; the model mentions it but its contract is out of scope (§12).
- **Assumption:** the implementation uses C# with an ORM (the model cites `Sale.Items`, `DbSet<Product>` and EF) on PostgreSQL 16 (§0, §3.2).

## 6. Consistency check

Final pass: the architecture was compared against the data model and against the requirements, product, domain and context documents.

### 6.1 Coverage of the model

| Check | Result | Source |
|---|---|---|
| The five entities (`Category`, `Product`, `Sale`, `SaleItem`, `User`) appear as components | OK | §2, §3 |
| The three aggregate roots are `Product`, `Sale` and `User`; `Category` is reference data | OK | §2.1 to §2.5 |
| The four foreign keys are covered: FK-1, FK-2 and FK-3 in the engine, FK-4 pending (T-12) | OK | §5 |
| Each rule has a mark (engine, domain-only or pending) | OK | §4 |
| Value objects `Money` and `Quantity` have no table | OK | §2, D-07 |
| The report has no table and is computed through a read port | OK | §1, §6.1 (Q9) |


### 6.2 Traceability to the other documents

| Architecture decision | Requirement that needs it | Domain rule | Source |
|---|---|---|---|
| Stock check in the engine (`ck_product_stock_non_negative`) | NFR-01, US-09 | `Product.Withdraw` | §2.2 |
| Optimistic concurrency with `xmin` | NFR-02 | Stock withdrawal is atomic with the sale line | §3, §6.1 (Q3) |
| Frozen name, price and category in `SaleItem` | NFR-04, US-11 | Frozen values on `SaleItem` | §2.4, §11.1 |
| Soft delete (`deleted_at`) | US-06, NFR-12 | Never physically deleted | §2.2, §7.1 |
| Hash port | US-01, NFR-08 | The domain never sees the plain password | §2.5, §9.2 |
| Read port for the report | US-11, NFR-09 | Report is not persisted | §1, §6.2 |
| Single currency, no audit columns | Out of scope in context and product | Closed decisions | §3, §8 |


### 6.3 Inconsistencies found in the model

| # | Finding | Effect on this architecture | Source |
|---|---|---|---|
| 1 | §3 says 22 columns, but the table lists 21 plus `xmin`, a system column | None; `xmin` is documented as the concurrency token | §3, §10.1 |
| 2 | §3 lists `category_name` among `product` columns, but it belongs to `sale_item` | Treated as a `sale_item` column | §2.4, §3 |
| 3 | §10 is a snapshot of 2026-09-19; §13 records fixes of 2026-09-20 (`deleted_at`, `sale_id NOT NULL`, unique `(sale_id, product_id)`, FK-3) | Section 3 follows §13 as the latest state | §10, §13 |
| 4 | `spec.md` CA-06.1 says "one row per product"; §11.1 can yield more than one | The report groups by the frozen category; the wording decision belongs to the owner | §11.1 |
| 5 | The sale line table marks `category_name` as pending (T-11) in §3, but as engine in §2.4 | Treated as pending until confirmed | §2.4, §3 |


### 6.4 Outcome

The architecture is consistent with the data model and with the other four documents. The open items are not architecture decisions: T-12 (sale authorship foreign key), T-13 (indexes), T-20 (moving domain-only rules into the engine) and the CA-06.1 wording (§4, §5, §6.2, §11.1).
