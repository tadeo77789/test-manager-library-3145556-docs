# Translation notes — from the Python/.NET spec to PHP (Laravel) + React

This document records **how the specification was translated** for the challenge stack. It is the evidence of the SDD work:
what was kept literally, what was translated, and which ambiguities were found and decided.

Base of the translation: [`../spec-python/`](../spec-python/). The .NET version (`../spec-.net/`) is said to describe the same business.
It was not used as a line-by-line cross-check in this translation; the business rules here come from the Python version.

Links to the unchanged documents: [data model](../spec-python/data-model.md) · [API contract](../spec-python/api-contract.md) ·
[ADRs](../spec-python/adr/) · [spec](../spec-python/spec.md) · [architecture overview](../spec-python/arquitectura-panoramica.md).

---

## 1. What is kept literally

| Area | Kept as written | Source |
|---|---|---|
| Actors and scope | Administrator, librarian, member (no login); out-of-scope list | [spec §1–§2](../spec-python/spec.md) |
| User stories and acceptance criteria | HU-01 … HU-10, CA-xx, RN-x, RNF | [spec §3](../spec-python/spec.md) |
| HTTP contract | Routes, `snake_case` fields, status codes, single error body, pagination `{items, page, size, total}` | [api-contract](../spec-python/api-contract.md) |
| Data model | Singular table names, schema `library`, UUID keys, `timestamptz`, no `DEFAULT`, no `status` / `days_late`, no `created_at` / `updated_at` | [data-model](../spec-python/data-model.md) |
| Categories | The five fixed categories with their fixed UUIDs | data-model §10 |
| Security | No secret in the repository, no `password` or `password_hash` in any response | constitution IX |
| Architecture rules | Ports and adapters, single composition point, one service per use case | constitution I–IV |

Note on the data model: Laravel's `$table->timestamps()` adds `created_at` / `updated_at` by default. The migrations must not use it.
The same applies to `$table->string(...)->default(...)` and to `$table->enum('status')`: none of them is allowed by the data model.

## 2. What is translated

| Python / .NET (spec) | PHP 8.3 + Laravel / React (this translation) |
|---|---|
| Python 3.12 / .NET | PHP 8.3 on Laravel; React with Vite on the frontend |
| FastAPI routers / controllers | Laravel routes in `routes/api.php` and controllers in `app/Adapters/Http` |
| Pydantic request schemas | Laravel form requests and validators in `app/Adapters/Http` |
| SQLAlchemy 2 models | Query Builder in `app/Adapters/Persistence` (no Eloquent in the domain or application layers) |
| Alembic revisions | Laravel migrations in `database/migrations` (ADR-001 is kept: `infra` has no DDL) |
| `dataclass(frozen=True)` / `record` | `readonly` classes and readonly properties in `App\Domain` |
| `import-linter` / architecture tests | deptrac rules or a PHPUnit scan in `tests/Architecture/` |
| pytest / xUnit | PHPUnit 11 |
| Argon2 via a library | `password_hash(..., PASSWORD_ARGON2ID)` and `password_verify` |
| PyJWT / JWT library | `firebase/php-jwt` (HS256) |
| Cover file storage | `CoverStorage` port with a local disk adapter |
| Time zone and clock | `Clock` port; `SystemClock` adapter; `CLOCK_OFFSET_DAYS` (see decision a) |
| Python package layout `src/library/` | Laravel layout `app/Domain`, `app/Application`, `app/Adapters`, `app/Bootstrap` |
| Composition module `bootstrap/composition/port_bindings.py` | `app/Bootstrap/PortBindingsServiceProvider.php` |
| Error language | Unchanged: the `message` field stays in Spanish (contract). Code and docs are in English |
| Server-side UI | React single-page app (`app`), talking to the API only over HTTP |

## 3. Traceability matrix (artifact → artifact)

| Spec element | Python / .NET location | Laravel / React location | Evidence |
|---|---|---|---|
| Constitution I (domain is pure) | `src/library/domain/` | `app/Domain/` | deptrac rule / architecture test |
| Constitution II (ports owned by the core) | `application/ports/outbound/` | `app/Application/Ports/` | architecture test |
| Constitution III (one composition point) | `bootstrap/composition/port_bindings.py` | `app/Bootstrap/PortBindingsServiceProvider.php` | architecture test |
| Constitution V (schema in migrations) | Alembic revisions | `database/migrations/` | `grep -ri "create table" infra/` is empty |
| `Book` aggregate (RN on copies) | `domain/book` | `App\Domain\Book` | T-04 |
| `Loan` aggregate (3 items, 14 days, state) | `domain/loan` | `App\Domain\Loan` | T-05 |
| `LendBooksService` | `application/services` | `App\Application\Services\LendBooksService` | T-09 |
| `ReturnItemService`, overdue | `application/services` | `App\Application\Services\ReturnItemService`, `OverdueService` | T-10 |
| Most-borrowed report (ADR-004) | `ReportReader` port | `App\Application\Ports\ReportReader` + Query Builder adapter | T-11, T-13 |
| Concurrency (ADR-002) | version token | conditional `UPDATE` on `available_copies` (decision c) | T-13 |
| Routes table (api-contract §3) | FastAPI routers | `routes/api.php` (prefix `/api/v1`) | T-16 |
| Single error body (api-contract §2) | exception handlers | exception renderer in `app/Adapters/Http` | T-16 |
| Login, token 8 h (HU-01) | `LoginService`, JWT | `LoginService`, `firebase/php-jwt` | T-07, T-14 |
| Initial admin (plan D-10) | startup hook | `PortBindingsServiceProvider` boot step or artisan command run at container start | T-17 |
| Demo data (T-21) | `tool/` | `tool/` (Node, calls the API) | T-21 |
| Screens (app) | — | `app/src/pages/*` | T-22 |
| Public site (T-23) | — | `page/` (static, no API) | T-23 |

## 4. Ambiguities found and decisions

### (a) Seeding three overdue loans "through the API"

The spec asks the demo seeder (`tool`) to create **three overdue loans through the API** (T-21). But the server sets `loaned_at` to the current time
when a loan is created. A loan created now cannot be 14 days late.

**Decision:** the `Clock` port has an adapter that adds `CLOCK_OFFSET_DAYS` (integer, default `0`) to the system time. The seeder runs the API with a negative offset
(for example `-15`), so the loans are stored 15 days in the past and are overdue for real. With the default value the system behaves normally.
The offset is read only by the clock adapter: the domain and the use cases still see a `Clock` and do not know about the offset.
The variable is documented in [`plan.md`](plan.md) §2.

### (b) Laravel default tables versus the `library` schema

Laravel ships migrations for `users`, `password_reset_tokens`, `sessions`, `cache` and `jobs` in the public schema.

**Decision:** the default migrations that the domain does not need are removed. All domain tables live in schema `library` (`library.user`, `library.book`, …).
The migration creates the schema first, and the database connection uses `search_path` `library`. Authentication is not Laravel's `users` table: it is the
`library.user` table behind the `UserRepository` port. This keeps the `infra` database empty until the API migrates it, as constitution V requires.

### (c) ADR-002 says "version token", but the data model has no `version` column

[ADR-002](../spec-python/adr/adr-002-concurrencia-optimista.md) uses a version token per book to detect the last-copy race. The data model has no `version` column,
and the data model is the reference (constitution X).

**Decision:** the optimistic check is done on the copy count itself. The loan adapter runs
`UPDATE library.book SET available_copies = available_copies - 1 WHERE id = ? AND available_copies = ?` with the value read before, and checks that exactly one row
changed; if zero rows changed, the request gets a 409. A `CHECK (available_copies >= 0 AND available_copies <= total_copies)` guarantees the range even if the
update logic has a bug. This keeps the data model unchanged and still satisfies RNF-2 (two loans for the last copy: one 201, one 409).

### (d) `GET /books/{id}/cover` is cited in the text but missing from the endpoint table

The API contract mentions the cover download in the text but does not list the route in the endpoints table.

**Decision:** the route is implemented (`GET /api/v1/books/{id}/cover`, returns the image or 404), because the cover is part of HU-03 (CA-03.4) and a
cover that cannot be downloaded is not usable. Upload and removal are the routes of the table (`PUT` and `DELETE /books/{id}/cover`).

### (e) Route order: `/loans/overdue` versus `/loans/{id}`

Both routes match `GET /api/v1/loans/...`. If `/loans/{id}` is registered first and has no pattern, the string `overdue` is captured as `{id}` and the request
fails with the id error path (404 or 422) instead of returning the overdue list.

**Decision:** `GET /loans/overdue` is registered before `GET /loans/{id}` in `routes/api.php`, and `{id}` is constrained to UUID format with a route pattern.
A contract test covers both routes.

### (f) Which spec is the base

The two versions describe the same product: the same stories, rules, endpoints and data model. Only the technology decisions differ.

**Decision:** `spec-python/` is the base, because the project brief names it as the reference version and its ADRs and `tasks.md` are the ones this
translation follows. The task brief states the two versions describe the same product. Where they differ on technology, the Python choice is translated to
Laravel/React and the .NET choice is ignored. Any business difference found later must be recorded here.

### (g) Language

The project owner decided that code, documentation, commits and READMEs are in **English**. The HTTP error `message` values and the texts that the librarian sees
in the interface stay in **Spanish**, because the contract (api-contract §1) and constitution XI require them.
This document and the rest of `spec-laravel/` follow the English decision.

---

## 5. Status

The translation covers the constitution, plan, architecture and tasks ([`constitution.md`](constitution.md), [`plan.md`](plan.md),
[`architecture.md`](architecture.md), [`tasks.md`](tasks.md)). Task status is tracked in `tasks.md`, which starts with every task **pending**.
