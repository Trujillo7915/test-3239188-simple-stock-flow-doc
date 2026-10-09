# Definition of Done (DoD) — Simple Stock Flow

> **A User Story or Task is DONE only when it satisfies EVERY item on this checklist.**  
> If any item is missing, the story remains in progress.

---

## 1. Mandatory Checklist

### 1.1 Code Quality & Architecture
- [ ] Code implements 100% of the User Story acceptance criteria.
- [ ] Follows Hexagonal Architecture principles (Domain has zero dependencies on EF Core or ASP.NET Core).
- [ ] Domain models use singular names (`Product`, `Sale`, `User`) and entity collections are typed (`§0`).
- [ ] No raw passwords, connection strings, or secrets exposed in code or commits (`§1`, `D-09`).
- [ ] Code passes static analysis (no compiler warnings, style rules enforced).

### 1.2 Testing & Invariants
- [ ] Unit tests cover all domain invariants (`price > 0`, `stock >= 0`, `quantity > 0`, username normalization).
- [ ] Integration tests verify atomic transaction rollback (e.g., sale fails if any product has insufficient stock).
- [ ] Concurrency tests verify optimistic locking via `xmin` on `product` table (`D-04`, `T-10`).
- [ ] 100% of test suite passes locally and in CI.

### 1.3 Database & Migrations (`ADR-001`, `spec/data-model.md`)
- [ ] Database modifications are implemented strictly via an EF Core migration file.
- [ ] Table names are singular in PostgreSQL `sales` schema (`category`, `product`, `sale`, `sale_item`, `user`) (`§0`).
- [ ] No column has a database `DEFAULT` value unless formally specified in an ADR (`§3`).
- [ ] All foreign keys specify explicit deletion behavior (`ON DELETE RESTRICT` for FK-1, FK-3, FK-4; `ON DELETE CASCADE` for FK-2) (`§5`).
- [ ] Schema passes the verification queries specified in `spec/data-model.md` §10.

### 1.4 Documentation (SDD Compliance)
- [ ] Corresponding SDD folders (`01-context`, `02-domain`, `03-product`, `04-requirements`, `05-architecture`) updated.
- [ ] Every functional or schema statement references its section in `spec/data-model.md` (e.g., `§2.2`, `FK-1`, `T-11`) or is explicitly marked as `[Supuesto]`.
- [ ] OpenAPI contracts updated in `07-api/` if API endpoints were added or modified.

### 1.5 Version Control & CI/CD
- [ ] PR created with Conventional Commit messages in English (`git-conventions.md`).
- [ ] PR reviewed and approved by at least 1 peer or Tech Lead.
- [ ] Branch merges cleanly without merge conflicts into `dev`.

---

## 2. Permitted Exceptions

Any temporary waiver must be formally authorized by the Tech Lead:
- Integration tests deferred due to environment setup: Must be logged with an explicit technical debt task.
- Unreleased foreign keys (e.g., pending `T-12` for `sold_by_user_id`): Must be documented in the technical debt log (`spec/data-model.md` §13).

---

## 3. What is NOT Done
- ❌ *"It compiles on my machine"* → It must pass CI and integration tests.
- ❌ *"The feature works, but the migration isn't generated"* → Schema must be versioned.
- ❌ *"I will document it next sprint"* → Code without up-to-date SDD documentation is not Done.
