# Requirements — Simple Stock Flow

> **Single source:** `spec/data-model.md`. Citations such as §x.y and Qn refer to that document.
> Anything that does not come from the model is marked as **Assumption**.

## 1. Actors

| Actor | Description | Source |
|---|---|---|
| Administrator (`admin`) | Internal operator with high privileges | §1, §2.5 |
| Seller (`seller`) | Internal operator who registers sales | §1, §2.5 |

There is no "customer" or "buyer" actor (§1).
**Assumption:** exactly what each role can do (the model only says there are two roles and that the `admin` creates sellers, §11 H-3).

## 2. User stories

| ID | Story | Rules behind it | Source |
|---|---|---|---|
| US-01 | As an operator, I want to log in with my username and password so I can use the system. | Unique, lowercase username; the password is only checked through the hash | §2.5, §6.1 (Q10) |
| US-02 | As an administrator, I want to create sellers so they can register sales. | The `admin` role is not granted at runtime: deployment provisions it | §11 (H-3, DP-04) |
| US-03 | As an operator, I want to search products by text and category so I can find them quickly. | Active products only, ordered by name, paginated | §6.1 (Q1) |
| US-04 | As an operator, I want to see the available categories to classify products. | Five fixed, read-only categories | §2.1, §9.1 (Q4) |
| US-05 | As an operator, I want to create and edit products (name, price, stock, category) to maintain the catalog. | `price > 0`; name required and trimmed; category required and existing | §2.2 |
| US-06 | As an operator, I want to deactivate a product without losing its history. | Soft delete with `deleted_at`; never physically deleted | §2.2, §7.1 |
| US-07 | As an operator, I want to attach an image to a product. | Only an opaque key is stored; no image means `NULL`, never an empty string | §1, §2.2 |
| US-08 | As a seller, I want to register a sale with several lines (product and quantity). | At least one line; a product cannot repeat within a sale; `quantity > 0` | §2.3, §2.4 |
| US-09 | As a seller, I want stock to be decremented when a line is added and to be blocked from selling more than what is available. | Withdrawing stock and adding the line is a single operation; `stock >= 0` | §2.2, §2.3 |
| US-10 | As an operator, I want to list sales within a date range. | Ordered by date descending, paginated | §6.1 (Q7) |
| US-11 | As an administrator, I want a per-product sales report for a date range. | Groups by product and frozen category; the end of the range cannot precede the start | §1, §6.1 (Q9), §11.1 |

## 3. Non-functional requirements

| ID | Category | Requirement | Source |
|---|---|---|---|
| NFR-01 | Integrity | Stock is never negative; the engine enforces it as the last barrier | §2.2 (`ck_product_stock_non_negative`) |
| NFR-02 | Concurrency | Simultaneous sales on the same product must not oversell; `xmin` is the concurrency token | §3 (`xmin`), §6.1 (Q3) |
| NFR-03 | Immutability | A registered sale is neither edited nor deleted; no operation allows it | §2.3, §7.1 |
| NFR-04 | Historical traceability | The sale line stores a frozen copy of name, price and category, so a closed period's report never changes | §1, §2.4, §11.1 |
| NFR-05 | Monetary accuracy | Amounts in `numeric(18,2)` with rounding to 2 decimals (`AwayFromZero`); single currency | §2.2, §3 |
| NFR-06 | Computation | Total and subtotal are computed, not stored | §1 |
| NFR-07 | Time | All timestamps are `timestamptz`; the server runs in UTC | §3 |
| NFR-08 | Privacy | The password hash never appears in logs, responses or errors and is never indexed; the report is not broken down by seller | §7, §7.1 (DP-02) |
| NFR-09 | Performance | Search (Q1), sales listing (Q7) and the report (Q9) use dedicated indexes; the report is computed in the engine | §6.1, §6.2 |
| NFR-10 | Maintainability | All DDL is defined through migrations; the engine prevails over the document if they contradict | Header "Rige bajo", §3.2 |
| NFR-11 | Referential integrity | `ON DELETE` policies defined: FK-1, FK-3 and FK-4 are `RESTRICT`; FK-2 is `CASCADE` | §5 |
| NFR-12 | Retention | Sales and lines are kept indefinitely; the only data physically deleted is the image binary | §7.1 |

## 4. Out of scope
No customers, payments, multi-currency, `created_at`/`updated_at` auditing, or category maintenance (§1, §2.1, §3, §8).

## 5. Pending items in the model that affect requirements
- `sale.sold_by_user_id` and FK-4: pending (T-12), §5.
- Search and report indexes: pending (T-13), §6.2.
- `CHECK` constraints for `price > 0`, `quantity > 0`, role and non-empty name: domain-only today (T-20), §4.
- **Possible contradiction:** `spec.md` CA-06.1 says "one row per product", but §11.1 produces more than one when a product was recategorized. Pending decision by the owner.
