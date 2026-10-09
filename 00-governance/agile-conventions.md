# Agile Team Conventions — Simple Stock Flow

> **Defines the agile framework and iterative engineering cadence for the project team.**

---

## 1. Sprint Cadence & Structure

| Dimension | Specification |
|---|---|
| **Sprint Length** | 2 weeks |
| **Sprint Start** | Monday 08:00 AM (Colombia Time, UTC-5) |
| **Sprint End** | Friday 05:00 PM (Colombia Time, UTC-5) |
| **Backlog Methodology** | Scrum with Continuous Delivery practices |
| **Primary Artifact** | SDD Documentation & Executable .NET / PostgreSQL increments |

---

## 2. Scrum Ceremonies

### 2.1 Sprint Planning
* **When:** Monday (Day 1), 08:00 AM – 10:00 AM.
* **Duration:** 2 hours maximum.
* **Attendees:** Full Development Team, Product Owner, Tech Lead.
* **Objective:**
  - Review stories meeting the **Definition of Ready (DoR)**.
  - Break stories down into tasks (Domain, Adapters, EF Migrations, SDD Docs).
  - Commit to Sprint Goal.

### 2.2 Daily Stand-up
* **When:** Monday through Friday, 05:00 PM (Colombia Time).
* **Duration:** 15 minutes strict timebox.
* **Format:**
  1. What did I achieve yesterday towards the sprint goal?
  2. What will I achieve today?
  3. Are there any blockers or domain/schema ambiguities?
* **Golden Rule:** Architectural discussions happen *after* the daily in a breakout session.

### 2.3 Backlog Refinement
* **When:** Thursday (Week 1), 03:00 PM – 04:30 PM.
* **Objective:** Groom user stories for the upcoming sprint, trace acceptance criteria to `spec/data-model.md`, and estimate effort.

### 2.4 Sprint Review
* **When:** Friday (Last Day), 03:00 PM – 04:00 PM.
* **Objective:** Demonstrate working increments against acceptance criteria and verify SDD consistency.

### 2.5 Sprint Retrospective
* **When:** Friday (Last Day), 04:00 PM – 04:45 PM.
* **Technique:** Plus/Delta or Start-Stop-Continue.
* **Outcome:** Actionable engineering improvements assigned to specific owners.

---

## 3. Estimation & Story Points

Story points follow the modified Fibonacci sequence (1, 2, 3, 5, 8, 13) using **Planning Poker**:

| Points | Complexity & Scope | Example Task in Simple Stock Flow |
|---|---|---|
| **1** | Trivial task (few hours) | Update error message formatting, seed category label correction (`§3.2`) |
| **2** | Small task (1 day) | Implement Value Object validation (`Money`, `Quantity`) |
| **3** | Medium task (2-3 days) | Implement `ISearchProductsUseCase` with EF Core query filters |
| **5** | Complex task (almost full sprint) | Atomic sale registration with stock deduction and optimistic locking (`xmin`) |
| **8** | High complexity / Needs split | End-to-end report aggregation engine with covering index |
| **13** | Epic | Must be split into sub-stories before entering a sprint |

---

## 4. Backlog Board & Workflow States

**Tool:** GitHub Projects / Azure DevOps

```
[ Backlog ] ──► [ Ready (DoR) ] ──► [ In Progress ] ──► [ In Review (PR) ] ──► [ Done (DoD) ]
```

| Column | Definition & Criteria |
|---|---|
| **Backlog** | Identified user stories pending technical refinement. |
| **Ready** | Story fulfills the **Definition of Ready** (acceptance criteria, schema citations). |
| **In Progress** | Assigned developer is actively implementing code or documentation. |
| **In Review** | PR created with green CI, under peer review. |
| **Done** | Fulfills the **Definition of Done** (code approved, tests passing, SDD updated). |
