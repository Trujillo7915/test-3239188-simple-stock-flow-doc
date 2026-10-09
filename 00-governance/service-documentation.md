# Module & Service Documentation Standard — Simple Stock Flow

> **Specifies required documentation standards for each functional module, API contract, and operational runbook in Simple Stock Flow.**

---

## 1. Modular Architecture Overview

In Simple Stock Flow's Hexagonal Architecture, core functionality is organized into cohesive application modules around the PostgreSQL `sales` schema:

```
src/
├── Domain/                  # Pure C# domain entities, value objects, domain events
├── Application/             # Use cases, inbound ports, DTOs
│   ├── Catalog/             # Products & Categories management
│   ├── Sales/               # Atomic sale registration & history
│   ├── Identity/            # Authentication & operator management
│   └── Reporting/           # Optimized read-model aggregations (Q9)
├── Adapters/
│   ├── Inbound/Api/         # REST Controllers & OpenAPI endpoints
│   └── Outbound/            # Persistence (EF Core), Cryptography, Image Storage
└── Infrastructure/          # Database migrations, Docker, deployment manifests
```

---

## 2. Required Documentation per Module

Every functional module MUST maintain up-to-date documentation aligned with the SDD directory structure:

| Document | Location | Purpose & Minimum Content |
|---|---|---|
| **Domain Specification** | `02-domain/` | Entity boundaries, invariants, value objects (`Money`, `Quantity`), domain events. |
| **Requirements Specification** | `04-requirements/` | User stories, Gherkin acceptance criteria, NFRs, traceability matrix. |
| **Architectural Mappings** | `05-architecture/` | Inbound/Outbound port interfaces, concurrency strategy (`xmin`), rule location. |
| **Physical Schema** | `spec/data-model.md` | Authoritative definition of tables, columns, constraints, indexes, verification queries. |
| **OpenAPI Contract** | `07-api/contracts/openapi/simple-stock-flow.yaml` | Machine-readable REST API specifications (OpenAPI 3.0). |

---

## 3. OpenAPI Contract Standard (API-First Principle)

1. **API-First Rule:** Before implementing a new REST endpoint, the request/response schema must be defined in the OpenAPI contract.
2. **Schema Conventions:**
   - Resource paths use lowercase plural nouns: `/api/v1/products`, `/api/v1/sales`, `/api/v1/categories`.
   - Request and response property names use `camelCase`.
   - Currency representation: Monetary values represented as standard decimals, no currency parameters accepted (`D-05`).
   - Standard Error Schema: All `4xx` and `5xx` responses must match `ApiErrorResponse` (`code`, `message`, `traceId`).

---

## 4. Runbook & Operational Procedures

Operational instructions must be maintained to ensure repeatable deployments and database verification:

### 4.1 Running Schema Migrations (EF Core)
```bash
# Apply migrations to PostgreSQL container
dotnet ef database update --project src/Adapters/Outbound/Persistence/
```

### 4.2 Verifying Database Constraints & Columns
Execute the queries from `spec/data-model.md` §10:
```bash
docker compose exec -T db psql -U simple_stock_flow -d simple_stock_flow -c \
  "SELECT table_name, column_name, data_type, is_nullable FROM information_schema.columns WHERE table_schema = 'sales';"
```
Must return exactly **21 rows** with no database defaults (`§10.1`).
