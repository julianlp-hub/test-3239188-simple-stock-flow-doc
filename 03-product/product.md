# Product — Simple Stock Flow

> **Single source:** `spec/data-model.md`. Citations such as §x.y refer to that document.
> Anything that does not come from the model is marked as **Assumption**.

## 1. Problem it solves

A business that sells products needs to know **how many units it has**, **what was sold, by whom and at what price**, and be able to **add up sales per product over a period**. Without a system, stock and sales drift apart: items that no longer exist are sold, the selling price is lost, and reports change when the catalog changes.

The data model shows which problems are solved by design:

| Business problem | How the model solves it | Source |
|---|---|---|
| Selling more than what is in stock | `stock >= 0` enforced by the engine and stock withdrawal that is atomic with the sale line | §2.2, §2.3 |
| Losing the selling price when a product changes | Name, price and category are frozen in the sale line | §1, §2.4 |
| Reports that change over time | The report groups by the frozen value | §11.1 |
| Losing history when a product is "deleted" | Soft delete, never physical deletion | §2.2, §7.1 |
| Not knowing who sold | Each sale records the operator who made it | §2.3 |

**Assumption:** the business is a small, in-person retail shop selling hardware and supplies. The model does not say so explicitly, but it is suggested by the five seeded categories (General, Tools, Electricity, Plumbing, Paint, §9.1) and by the absence of customers and payments (§1, §7).

## 2. Product vision

> *For* small businesses that sell items from a fixed catalog, *Simple Stock Flow* is an inventory and sales system that keeps stock correct and preserves a sales history that does not change, *unlike* tracking everything by hand or in loose spreadsheets.

## 3. Users

| User | What they need | Source |
|---|---|---|
| Seller (`seller`) | Register sales and look up products | §2.5 |
| Administrator (`admin`) | Create sellers, maintain the catalog and view reports | §2.5, §11 (H-3) |

Buyers are not users of the system (§1).

## 4. Value proposition

1. **Reliable stock:** it never drops below zero and is decremented together with the sale (§2.2, §2.3).
2. **Stable history:** sales are immutable and keep a copy of what was sold (§2.3, §2.4).
3. **Per-product report over a date range,** computed in the engine (§1, §6.1 Q9).
4. **Minimal privacy footprint:** only the operator is stored, not the end customer (§7).

## 5. What the product is and is not

**It is:** a product catalog with five fixed categories, sales registration, internal users with two roles, and a per-product report.

**It is not** (decisions closed in the model):
- A customer or payment system (§1, §7).
- A multi-currency system (§3).
- A system with change auditing of the catalog (§8).
- A category manager (§2.1, §4.1).
- A system with extra product attributes such as description or SKU (§1, DP-03).
- A per-seller report (§7.1, DP-02).

## 6. Success criteria

**Assumption:** the model defines no success metrics. These are derived from its rules:

- No operation leaves a product with negative stock.
- A report for a closed range always returns the same result.
- No sale exists without at least one line.
- A seller can register a multi-line sale in a single operation, and if any line exceeds the available stock the operation fails without leaving stock negative (§2.2, §2.3).

## 7. Known risks and debts in the model

- Several rules live only in the domain (`price > 0`, `quantity > 0`, valid role); a manual `INSERT` bypasses them (§4, T-20).
- Sale authorship is plain text, with no foreign key to `user`, until T-12 (§5, FK-4).
- `spec.md` CA-06.1 ("one row per product") conflicts with the decision in §11.1 (there may be more than one row per product after a recategorization).

## 8. Observations on model fit

**Assumption / team observation:** when contrasting the model with an electric-motorbike shop known to the team, two limits appear that the model declares deliberately:

- Categories are five, fixed and cannot be created (§2.1, §4.1), so there is no dedicated category for vehicles.
- A product has only name, price, stock, category and image (§1, DP-03), with no serial or chassis number.

These are closed decisions of the model, not errors; they are noted so that anyone adopting the system in another kind of business knows where it does not fit.
