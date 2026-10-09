# Documentation Rules (SDD Standards) — Simple Stock Flow

> **Guidelines for authoring, updating, and maintaining Specification-Driven Development (SDD) documentation.**

---

## 1. Core Principle

> **"Documentation is code. If it is not up to date, it is broken."**

In Simple Stock Flow, documentation is not an afterthought written post-implementation. It is the primary engineering specification that drives the code. Any pull request that alters domain logic, database structure, or API behavior **must** update the corresponding documentation files in the same PR.

---

## 2. Language Policy

| Artifact Category | Standard Language | Enforcement |
|---|---|---|
| C# Source Code (Entities, Interfaces, Use Cases) | **English** | CI compiler & style rules |
| Code Comments & Docstrings | **English** | Code Review |
| Git Commits & Branch Names | **English** (Conventional Commits) | PR Gate |
| SDD Specifications (`01` to `05`) | **Spanish** (or Technical English) | Consistent per section |
| Database Catalog Names (Tables, Columns, Indexes) | **English** (ASCII, singular for tables `§0`) | EF Core migrations |
| OpenAPI 3.0 API Contracts | **English** | Contract validation |

---

## 3. Directory Layout & Document Hierarchy

```
test-3239188-simple-stock-flow-doc/
├── 00-governance/         # Team agreements, Git, DoD, DoR, Security rules
├── 01-context/            # System context, C4 Level 1 diagram, in/out scope
├── 02-domain/             # Ubiquitous language, entities, VOs, domain events
├── 03-product/            # Problem framing, product vision, core principles
├── 04-requirements/       # User stories, Gherkin criteria, NFRs, traceability matrix
├── 05-architecture/       # Hexagonal architecture, aggregates, ports, verification loop
├── spec/data-model.md     # Single source of truth for the physical data model (06-data)
└── README.md              # Project overview and evaluation instructions
```

---

## 4. Traceability & Citation Rules

1. **Every business claim must cite its source:** When describing an entity, rule, constraint, or query, reference the exact section in `spec/data-model.md`:
   - Examples: `[§2.2]`, `[FK-1]`, `[ck_product_stock_non_negative]`, `[ADR-002]`, `[D-05]`, `[T-10]`.
2. **Explicit Assumptions (`[Supuesto]`):** If an architecture detail, requirement, or domain rule is derived through logical deduction but is not explicitly declared in `spec/data-model.md`, it **must** be marked with the `[Supuesto]` tag and registered in the section's assumption table.
3. **No Phantom Columns:** Never invent database columns or relationships not backed by the 22 physical columns of `spec/data-model.md` §3.

---

## 5. Formatting Standards

- **Headings:** Use `# H1` for document title, `## H2` for main sections, and `### H3` for subsections. Do not nest deeper than `H3`.
- **Diagrams:** Use **Mermaid** for all architectural, entity-relationship, and sequence diagrams (`flowchart`, `classDiagram`, `erDiagram`, `sequenceDiagram`).
- **Tables:** Use Markdown tables for matrices, catalogs, and comparative registers.
- **Code Snippets:** Specify the syntax highlighting language tag in code blocks (`csharp`, `sql`, `bash`, `markdown`).
