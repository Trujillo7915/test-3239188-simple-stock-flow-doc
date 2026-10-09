# Git Conventions — Simple Stock Flow

> **Mandatory rules for all commits and branches in `simple-stock-flow`.**

---

## 1. Branch Strategy

Simple Stock Flow utilizes a GitFlow-inspired trunk-based branch model:

```
main        ← Production/Final release. Always stable and deployable.
  └── dev   ← Integration branch for active sprint development.
        ├── feat/[feature-name]    ← One branch per user story / feature slice
        ├── fix/[bug-description]  ← Bugfixes for staging or integration issues
        ├── docs/[topic-name]      ← SDD documentation updates
        ├── chore/[task-name]      ← Migrations (EF Core), tooling, CI workflows
        └── hotfix/[urgent-fix]    ← Critical patches merged directly to main
```

### Core Rules:
1. **No direct commits to `main` or `dev`:** All changes enter via Pull Requests.
2. **One task = One branch:** Do not bundle unrelated changes or multiple user stories into a single branch.
3. **Atomic scope:** A branch must address a single concern (e.g., `feat/atomic-sale-registration`, `docs/04-requirements-spec`).
4. **Branch deletion:** Feature branches must be deleted immediately after a successful merge.

---

## 2. Branch Naming Format

Branch names must be written in **lowercase English** using kebab-case:

```
[type]/[kebab-case-description]
```

### Examples:
- `feat/product-stock-optimistic-locking`
- `feat/frozen-sale-items-projection`
- `fix/category-name-unique-constraint`
- `docs/domain-ubiquitous-language`
- `chore/ef-migration-rename-to-singular`
- `hotfix/prevent-negative-stock-overflow`

---

## 3. Commit Format (Conventional Commits in English)

Every commit message in this project MUST be written in **English** following the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
[type]([scope]): [short imperative description in lowercase, no trailing period]

[optional body — explain WHY the change was made and technical trade-offs]

[optional footer — reference task ID, user story, or spec section]
```

### Allowed Commit Types:

| Type | When to use | Example |
|---|---|---|
| `feat` | New domain feature or API capability | `feat(sales): implement atomic sale registration with stock deduction` |
| `fix` | Bug fix or invariant patch | `fix(catalog): enforce non-negative check on stock withdrawal` |
| `docs` | Documentation only (SDD folders) | `docs(02-domain): update ubiquitous language and domain events` |
| `refactor`| Code restructuring without changing behavior | `refactor(persistence): map xmin system column to shadow property` |
| `test` | Adding or updating unit, integration tests | `test(sales): add concurrency test for simultaneous stock decrement` |
| `chore` | EF Core migrations, build tools, dependencies | `chore(db): add EF Core migration for singular table naming` |
| `perf` | Database index or query performance | `perf(reports): add covering index on sale_item for index-only scans` |

### Concrete Examples for Simple Stock Flow:
```git
feat(sale): enforce single product per sale invariant

Ensure duplicate products cannot be added to the same sale.
Aligns with constraint IX_sale_item_sale_id_product_id.

Refs: spec/data-model.md §2.3, T-20
```

```git
docs(05-architecture): add closure section and consistency verification loop

Validate that all Hexagonal components and aggregates match
the 22 physical columns and 8 database constraints.
```

---

## 4. Pull Request (PR) Policy

1. **Size Limit:** A PR must not exceed 400 lines of code (excluding auto-generated EF Core migration snapshots and tests).
2. **Code Review:** Requires at least 1 approval from a peer or Tech Lead.
3. **Automated Validation:**
   - Unit and integration tests must pass.
   - EF Core migrations must apply cleanly without manual schema divergence (`ADR-001`).
   - Linting and compilation warnings treated as errors.
4. **Documentation Sync:** Any PR that modifies business logic, domain entities, or database schema must update the corresponding SDD files (`01` to `05`) within the same pull request.

---

## 5. Merge Policy

- **Squash and Merge:** Required for merging `feat/*`, `fix/*`, and `docs/*` into `dev` to maintain a linear and legible history.
- **Merge Commit:** Used for releasing `dev` into `main` to preserve release milestones.
- **Fast-Forward / Rebase:** Prohibited on shared public branches.
