# Laravel + React decisions — Manager Library

How the spec (written for Python and .NET) was read, interpreted and carried over to **Laravel 13 (PHP 8.3) + React**.
Business rules, acceptance criteria, data model and API contract are kept as written; technology choices are translated.

Read in this order: this file → [ADR-005](adr-005-optimistic-control-without-version-column.md) → the `api` README (evidence per task).

## 1. Reading the two spec versions

`spec-python/` and `spec-.net/` were diffed line by line. They differ **only** in technology
(FastAPI vs ASP.NET, Alembic vs EF migrations, Argon2id vs BCrypt, import-linter vs NetArchTest, pytest vs xUnit).
Stories, rules, endpoints and data model are identical, so the traps are not in the difference between versions:
they are internal contradictions of the spec and Laravel defaults that clash with it.

## 2. Stack translation

| Spec | Laravel / React | Why |
|---|---|---|
| FastAPI / ASP.NET Minimal API | Laravel routes + thin controllers in `app/Adapters/Inbound/Http.php` | Contract in `api-contract.md` |
| SQLAlchemy / EF Core | Query builder, **no Eloquent** | `HasUuids` produces UUID v7 (contract: v4) and models expect `created_at/updated_at` (data model: none) |
| Alembic / EF migrations | Laravel migrations written as explicit SQL | What runs is exactly `data-model.md` §3–§5 |
| `dataclass(frozen)` / `record` domain | Plain PHP classes in `app/Domain` | Article I: no framework in the domain |
| `port_bindings.py` / `PortBindings.cs` | `app/Bootstrap/PortBindings.php` (service provider) | Article III: single composition point |
| Argon2id / BCrypt | Argon2id (`password_hash`) | Python version chosen; both satisfy art. IX |
| JWT HS256, 8 h | Own minimal HS256 issuer behind `TokenIssuer` | Stateless; no Sanctum (it adds a token table) |
| pytest / xUnit + import-linter / NetArchTest | PHPUnit + `ArchitectureTest` scanning the source | Same evidence for T-19 |
| React frontend | React (Vite) | As required by the challenge |

## 3. Traps found and how each was resolved

| # | Trap | Where in the spec | Resolution |
|---|---|---|---|
| 1 | ADR-002 asks for a **version token** on `book`, but `data-model.md` has no `version` column and says "if something contradicts this document, this document wins" | ADR-002 vs data-model §3 | No column invented. The counter itself is the token: `UPDATE … SET available_copies = available_copies - 1 WHERE id = ? AND available_copies >= 1`. 0 rows → 409 `BOOK_NOT_AVAILABLE`. See [ADR-005](adr-005-optimistic-control-without-version-column.md) |
| 2 | The spec contradicts itself on who wins: data-model says "this document wins", constitution art. X says "the engine wins, fix the document" | data-model header vs constitution X | The constitution governs (data-model "rige bajo constitution.md"). Applied to T-18: engine changed, then `data-model.md` §4 corrected |
| 3 | `ck_book_total_positive` and `ck_user_role` are **pending (T-18)**; putting them in the initial migration would make §4 lie | data-model §4, tasks T-02/T-18 | Separate migration `2026_10_13_000002`; §4 updated in both spec folders |
| 4 | `GET /books/{id}/cover` is only mentioned in prose, missing from the §3.4 table | api-contract §3.4 | Implemented (authenticated); the frontend needs it to render covers |
| 5 | `PUT /books/{id}` replaces title, author, category and total: **ISBN is not editable** | api-contract §3.4 | Sending a different ISBN → 422 |
| 6 | The seeder must use **the API** and leave **exactly 3 overdue** lines, but `due_at = loaned_at + 14 d` and `loaned_at` is set by the server | tasks T-21, data-model §9 | The `Clock` port has a `CLOCK_OFFSET_DAYS` setting used only by a seed-only API instance; no backdoor endpoint, no SQL. See `tool` README |
| 7 | 15 loans, 10 members, max 3 unreturned per member, and "member with overdue cannot borrow" | T-21 + RN-2/RN-3 | Seeder plan orders loans, returns and the past-dated phase so every call is valid |
| 8 | Laravel ships `users`, `sessions`, `cache`, `jobs`, `password_reset_tokens` migrations (and Sanctum `personal_access_tokens`) | data-model §2 "six entities, six tables" | Removed; session/cache/queue drivers set to `array`/`sync`; Laravel's `migrations` table lives in `public`, outside `library` |
| 9 | `$table->id()` and `timestamps()` add DEFAULT and audit columns | data-model §3, §8 | Explicit SQL, uuid PKs, no DEFAULT; verified by a test reading `information_schema` |
| 10 | Laravel serializes dates as `…00.000000Z` | api-contract §1 | Formatted as `Y-m-d\TH:i:s\Z` |
| 11 | Laravel paginator returns `data/current_page/per_page` | api-contract §1 | `{items, page, size, total}` |
| 12 | Laravel's default errors (`{message, errors}`, HTML 404, redirect to `login` → 500) | RNF-4 | One exception renderer: every error is `{code, message, details}` |
| 13 | `/loans/overdue` would be captured by `/loans/{id}` | api-contract §3.6 | Declared first |
| 14 | `user` is a reserved word in PostgreSQL | data-model §0 | `library."user"` |
| 15 | PHP only parses `multipart/form-data` on POST; the contract uses **PUT** for the cover | api-contract §3.4 | Minimal multipart reader for PUT; type checked by file signature (finfo), not extension |
| 16 | Report `to` is an inclusive **date**; `loaned_at <= to` loses the last day | CA-10.2, api-contract §3.7 | `loaned_at >= from AND loaned_at < to + 1 day` |
| 17 | Report groups by **frozen** values: the same book can appear in two rows | ADR-004, CA-10.1 | `GROUP BY book_id, book_title, category_name` on `loan_item`; never joins the live catalog |
| 18 | Validation order when lending | architecture §4 | Overdue (409) → limit of 3 (422, current count in `details`) → copies (409), all in one transaction |
| 19 | A double return must never restock twice, even concurrently | CA-07.2 | `UPDATE … WHERE returned_at IS NULL`; 0 rows → 409 |
| 20 | 403 must not reveal whether a resource exists | api-contract §4 | Role middleware runs before the controller looks anything up |
| 21 | Login must not say whether user or password failed | CA-01.2 | Same message for both |
| 22 | Requesting role `admin` via API is **422**, not 403 | CA-02.3 | `VALIDATION_ERROR` on field `role` |
| 23 | Admin cannot be seeded by the DB (only the hash port can hash) | plan D-10, data-model §9 | `php artisan library:bootstrap-admin` at container start, from `ADMIN_USERNAME`/`ADMIN_PASSWORD`, idempotent |
| 24 | Cover removal order: null the key and commit **before** deleting the binary | data-model §7 | Implemented in that order |
| 25 | Concurrent loans for the same member could bypass the limit of 3 (domain-only rule) | data-model §4 | Member row locked (`FOR UPDATE`) inside the lending transaction: per member, not library-wide |

## 4. Gaps in the spec and the decision taken

| Gap | Decision |
|---|---|
| No error `code` for missing/expired token (only `INVALID_CREDENTIALS` is listed for 401) | `UNAUTHENTICATED`, same single body |
| `size` above 50 | 422 `VALIDATION_ERROR` (not silently clamped) |
| How "days late" is counted | Whole days elapsed since `due_at` (floor); sorted by `due_at` ascending = most late first |
| Whether `GET /loans/overdue` is paginated | Same `{items, page, size, total}` shape as every list |
| Lending a soft-deleted or unknown book / member | 404 `NOT_FOUND` with the offending field in `details` |
| 405 / 429 / 500 codes | `METHOD_NOT_ALLOWED`, `TOO_MANY_REQUESTS`, `INTERNAL_ERROR` with the single body |

## 5. Deliberate deviation

Constitution art. XI says documentation is in Spanish. The instructor required **everything in English** except a minimal part.
Applied: code, comments, docs and commits in English; **user-facing messages** (API error `message`, frontend labels) in Spanish,
as RN-6 and the contract example require. The existing spec files were edited in their own language (Spanish) to stay consistent.
