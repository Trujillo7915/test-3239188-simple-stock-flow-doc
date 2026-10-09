# Security Policy — Simple Stock Flow

> **Mandatory security baseline and operational governance for Simple Stock Flow.**

---

## 1. Security Principles

1. **Defense in Depth (`ADR-002`):** Invariants are defended both in the C# domain layer and as hard constraints in PostgreSQL (e.g., `ck_product_stock_non_negative`, `FK-1` to `FK-4`, unique indexes).
2. **Least Privilege (`§2.5`, `§3`):** Operators are granted only the permissions required for their specific role (`admin` vs. `seller`).
3. **Fail Secure:** Any transaction or operation that encounters an unexpected condition, stock shortfall, or constraint violation rolls back completely.
4. **Zero Trust & Domain Isolation (`D-09`):** The domain core never trusts external inputs and never handles raw passwords.
5. **Privacy by Design (`§7`, `DP-02`):** Attribute-by-attribute privacy classification; metrics that invade operator privacy are explicitly excluded.

---

## 2. Authentication & Credential Custody

### 2.1 Password Security (`D-09`, `§1`, `§2.5`)
- Raw passwords MUST NEVER be received, handled, or stored by domain entities.
- Password hashing is encapsulated in the `IPasswordHasher` outbound port using **BCrypt** (cost factor >= 12) or **Argon2id**.
- The resulting hash is stored in `sales.user.password_hash` (`varchar(512)`).
- **Absolute Rule (`§7`):** `password_hash` MUST NEVER appear in application logs, API response payloads, debug projections, or error traces.
- `password_hash` is **never indexed** in PostgreSQL (`§7`).

### 2.2 Operator Provisioning (`§9.2`, `DP-04`, `§11.1`)
- The initial `admin` account is **never seeded by database SQL scripts** (avoids hardcoding hashes or duplicating algorithms in SQL). It is provisioned on application bootstrap using environment variables (`§9.2`).
- The application exposes NO public anonymous registration endpoint.
- Only authenticated operators with `admin` role may register new `seller` operators (`DP-04`).

---

## 3. Authorization (Role-Based Access Control - RBAC)

Simple Stock Flow implements a closed two-role permission matrix (`§2.5`, `§3`):

| Role | Operational Scope | Authorized Actions & Endpoints |
|---|---|---|
| `admin` | Business Owner / Store Manager | • Create, edit, and deactivate catalog products (`HU-CAT-02`, `03`).<br>• Restock inventory (`Product.Restock`).<br>• Query aggregated sales reports by date range (`HU-REP-01`, `Q9`).<br>• Provision new `seller` user accounts (`DP-04`).<br>• Perform point-of-sale checkout (`HU-VTA-01`). |
| `seller` | Cashier / Sales Representative | • Search and list active catalog products (`HU-CAT-01`, `Q1`).<br>• Check individual product details (`Q2`).<br>• Register atomic sales with instant inventory decrement (`HU-VTA-01`).<br>• Retrieve receipts for completed sales (`HU-VTA-02`, `Q6`).<br>• *Forbidden:* Access to sales reports, catalog modification, user management. |

---

## 4. Privacy & Data Classification (Attribute by Attribute)

Derived strictly from `spec/data-model.md` §7:

| Table | Attribute | Classification | Required Security Handling | Retention Policy (`§7.1`) |
|---|---|---|---|---|
| `user` | `id` | Non-sensitive | Opaque UUID | Indefinite |
| `user` | `username` | **Personal Data (PII)** | Restricted internal access. Not in public APIs | Indefinite, no deletion |
| `user` | `password_hash` | **Authentication Secret** | **Never logged or returned**. Zero indexing | Indefinite, no historical versioning |
| `user` | `role` | Confidential | Internal access control token | Indefinite |
| `sale` | `sold_by` | **Personal Data** | Visible only on receipts/invoices. Restricted | **Indefinite. Never edited or deleted** |
| `sale` | `sold_by_user_id` | **Indirect Personal Data** | Foreign key to user (`FK-4`). Restricted | Indefinite (`T-12`) |
| `sale` | `sold_at` | Non-sensitive | UTC business timestamp | Indefinite |
| `sale_item`| all columns | Commercial data | Non-personal transaction details | Indefinite (with sale) |
| `product` | all columns | Commercial catalog | Public catalog information | **Soft delete only (`deleted_at`), never physical** |
| `category`| all columns | Public reference | Static reference data | Indefinite, no deletion |

### Customer Privacy Guarantee (`§1`, `§7`, `§12`)
- Simple Stock Flow collects **zero customer personal data**.
- No payment card data (PCI-DSS out of scope).
- No buyer names, emails, addresses, or identification numbers stored.

---

## 5. Binary Image Retention & Deletion Security (`§7.1`, `D-08`)

When replacing an image or soft-deleting a product:
1. **Step 1 (Database Transaction):** Set `product.image_key = NULL` and commit to PostgreSQL.
2. **Step 2 (Storage Port):** Invoke `IImageStorage.DeleteAsync(key)` to purge the binary file.
*Reasoning:* An orphaned binary in object storage is harmless; a dangling database key pointing to a missing binary causes broken UI images (`§7.1`).
