# Methodology — DDD, SDD and TDD

How the three approaches were applied in Manager Library.
In one line: **the domain models the business, the spec says what it must do, the tests prove it.**

| Approach | Question it answers | Where |
|---|---|---|
| DDD (Domain-Driven Design) | How is the code organized so it reflects the business? | `api/app/Domain` |
| SDD (Spec-Driven Development) | What must the system do, and how is it verified? | `docs/spec-*`, `laravel-decisions/` |
| TDD (Test-Driven Development) | Does each rule actually hold? | `api/tests` |

## 1. DDD — Domain-Driven Design

The code is organized around the library's business, not around the database or the framework.

- **Ubiquitous language:** member, loan, loan item, copy, overdue, frozen value. The same words are used
  in the spec, the code and the screens.
- **Aggregates** guard their own invariants:
  - `Book`: 0 <= available copies <= total, total >= 1, non-empty title.
  - `Loan`: 1 to 3 items, no repeated book, due in 14 days, immutable except for `returned_at`.
- **Value objects** validate themselves and have no identity: `Isbn` (13 digits), `Quantity` (>= 1),
  `DateRange` (end not before start), `Username` (trimmed, lowercase).
- **Computed state:** a loan's status (`open`, `overdue`, `closed`) is derived, never stored.
- **Hexagonal architecture:** the domain is plain PHP with no Laravel dependency (constitution art. I).
  The application declares ports (`Clock`, `PasswordHasher`, repositories) and adapters implement them
  (art. II), all wired in one composition point, `app/Bootstrap/PortBindings.php` (art. III).

## 2. SDD — Spec-Driven Development

The spec is the source of truth; code follows it, not the other way round.

- **Reading order:** constitution -> spec (user stories and acceptance criteria) -> plan and architecture
  -> data model and API contract -> tasks -> ADRs.
- **Translation:** business rules, contract and data model were kept as written; technology choices
  (FastAPI / ASP.NET, Alembic / EF) were translated to Laravel + React (`README.md` §2).
- **Contradictions found and resolved:** for example the version token from ADR-002 that the data model
  does not have (ADR-005), or the T-18 constraints that had to go in their own migration
  (`README.md` §3 lists 25 cases).
- **Evidence:** a task is done only when its evidence exists and runs (constitution art. XII).

## 3. TDD — Test-Driven Development

Each rule from the spec has an automated test that proves it.
Note: tests were written alongside the implementation, not strictly before it (no full red-green-refactor cycle).

| Test | What it proves | Spec |
|---|---|---|
| `tests/Unit/DomainTest.php` | Domain invariants with no database or framework | T-03..T-06 |
| `tests/Feature/ApiContractTest.php` | Contract, error codes, single error body, frozen report, real PostgreSQL | T-07..T-16, ADR-004 |
| `tests/Unit/ArchitectureTest.php` | Constitution articles I, II and III | T-19 |
| `tests/concurrency/last-copy.sh` | Last copy lent twice at once: one 201, one 409 | RNF-2, ADR-002 |