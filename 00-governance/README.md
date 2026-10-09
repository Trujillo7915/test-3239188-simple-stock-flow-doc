# 00-governance — Team Governance & Engineering Standards

> **Project:** Simple Stock Flow (`test-3239188-simple-stock-flow-doc`)  
> **Source Baseline:** ADSO Governance Framework  
> **Target System:** Simple Stock Flow (Point of Sale & Inventory System)

This directory defines the binding engineering rules, quality agreements, security policies, and workflow standards for the **Simple Stock Flow** project. Every team member and contributor must adhere to these conventions.

---

## Documents in this section

| File | Purpose | Key Project Alignment |
|---|---|---|
| [git-conventions.md](./git-conventions.md) | Branching strategy, Conventional Commits in English, PR lifecycle | Aligned with EF Core migration workflows and SDD tracking |
| [agile-conventions.md](./agile-conventions.md) | Sprint structure, ceremonies, estimation, and backlog management | Sprints focused on SDD reverse engineering and feature slices |
| [definition-of-done.md](./definition-of-done.md) | Criteria required for a User Story or Task to be considered Done | Mandatory verification of domain invariants and schema checks |
| [definition-of-ready.md](./definition-of-ready.md) | Quality gate for a User Story to enter an active sprint | Traceability to `spec/data-model.md` and explicit assumptions |
| [documentation-rules.md](./documentation-rules.md) | SDD authoring standards, file structure, ownership, and citation rules | Traceability to data model sections (`§1`, `§2`, `FK`, etc.) |
| [service-documentation.md](./service-documentation.md) | Required documentation per hexagonal module and API contract | Specification for core sales, catalog, and identity modules |
| [security-policy.md](./security-policy.md) | Security principles, RBAC (`admin`/`seller`), secret management, audit | Enforces `D-09` (password hash), `DP-02`, and data privacy |
| [security-rules.md](./security-rules.md) | Code-level technical security controls (OWASP Top 10, sanitization) | Protection of `user.password_hash`, input validation in C#/EF Core |

---

## How Governance Applies to Simple Stock Flow

Governance rules apply to **the entire codebase, specifications, and data models** of Simple Stock Flow:
1. **Schema Authority (`ADR-001`, `spec/data-model.md`):** All database changes must be executed strictly through Entity Framework Core migrations. No manual production DDL.
2. **Strict Domain Language (`§0`, `§1`):** English for all code symbols, tables in singular (`sales.category`, `sales.product`, `sales.sale`, `sales.sale_item`, `sales.user`), and Spanish/English domain documentation strictly separated.
3. **Traceability Rule:** Any functional statement, rule, or architecture component must cite its origin in `spec/data-model.md` or be declared as a `[Supuesto]` (Assumption).
4. **Conflict Resolution:** If a local implementation contradicts governance, this governance document governs unless an approved Architecture Decision Record (ADR) formally supersedes it.
