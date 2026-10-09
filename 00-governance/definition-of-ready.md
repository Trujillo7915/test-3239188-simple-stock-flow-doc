# Definition of Ready (DoR) — Simple Stock Flow

> **A User Story can enter an active Sprint ONLY when it satisfies all criteria on this checklist.**  
> If an item is missing, the story remains in the Backlog for further refinement.

---

## 1. Mandatory Checklist

### 1.1 Business Context & Story Definition
- [ ] User Story follows the standard persona template:
  > **As a** [Internal Role: `admin` | `seller` (`§1`, `§2.5`)]  
  > **I want to** [execute a business action]  
  > **So that** [achieve an operational outcome]
- [ ] Business value and operational priority are clearly understood.
- [ ] Fits within product boundaries: No customer profiling (`DP-02`), no secondary attributes (`DP-03`), monocurrency system (`D-05`).

### 1.2 Acceptance Criteria (AC)
- [ ] Acceptance criteria written in unambiguous format (Given-When-Then or clear bullet rules).
- [ ] All edge cases and negative validation paths defined (e.g., attempting to withdraw more stock than available, inserting non-positive prices, non-existent category).
- [ ] Invariant enforcement location is specified: whether it is enforced in **Domain**, in **Database Engine**, or marked as **Pending Task** (`§4`).

### 1.3 Data Model Traceability
- [ ] Physical tables and columns touched are mapped to `spec/data-model.md` §3.
- [ ] Applicable foreign key behaviors and constraints identified (`FK-1` to `FK-4`, `ck_product_stock_non_negative`, unique indexes).
- [ ] Query patterns and indexing requirements identified against Q1 to Q10 (`§6.1`).
- [ ] If a requirement does not originate in `spec/data-model.md`, it is explicitly cataloged as a `[Supuesto]`.

### 1.4 Technical Feasibility & Dependencies
- [ ] Concurrency requirements evaluated: specifies if `xmin` optimistic locking applies (`D-04`, `T-10`).
- [ ] Technical dependencies resolved: migrations, seed data (`§9.1`), or upstream use cases already in place.
- [ ] Security classification checked: does the story touch PII (`username`, `sold_by`) or secrets (`password_hash`)? (`§7`).

### 1.5 Estimation & Sizing
- [ ] Estimated by the team using Story Points (Planning Poker).
- [ ] Sized at **5 points or less**. If estimated at 8 or 13, it must be split prior to sprint planning.

---

## 2. Example: A Story Meeting the DoR

```markdown
### HU-VTA-01: Atomic Sale Registration with Inventory Deduction
* **As a:** Seller (`seller`) or Administrator (`admin`)
* **I want to:** Register a sale with one or multiple line items
* **So that:** The sale is recorded and physical stock is decreased simultaneously.

**Acceptance Criteria:**
- Given a sale with 2 items, when confirmed, then `sale` record, 2 `sale_item` rows, 
  and stock withdrawal execute in a single database transaction.
- If item 1 has stock 5 and requested 10, then the transaction fails with DomainException
  and neither stock nor sale records are created.
- Product name, unit price, and category name are frozen at the instant of the sale.

**Traceability:**
- Tables: `sales.sale`, `sales.sale_item`, `sales.product` (§3)
- Constraints: FK-2 (CASCADE), FK-3 (RESTRICT, T-20), ck_product_stock_non_negative
- Concurrency: Validates `xmin` on each product (D-04)
- Estimate: 5 Story Points
- Status: READY FOR SPRINT
```
