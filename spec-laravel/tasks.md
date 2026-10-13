# Tasks — Manager Library (PHP 8.3 / Laravel + React)

Translation of [`../spec-python/tasks.md`](../spec-python/tasks.md). Same tasks, same dependencies, same business evidence.
The evidence column is translated to the Laravel stack: PHPUnit tests, `php artisan` commands, `curl` calls, and browser checks.

Each task is finished when its **evidence** exists and can be executed (Article XII). The **Status** column starts as **pending** for every task.
The status is updated only when the evidence has been run.

| # | Task | Depends on | Evidence (Laravel / React) | Status |
|---|---|---|---|---|
| T-01 | Project skeleton and `GET /health` | — | `curl localhost:8000/api/v1/health` returns `{"status":"ok"}` (Feature test); the container starts with one command | pending |
| T-02 | Initial migration: six tables, FKs, unique indexes, and the five categories seeded | T-01 | The queries of [`../spec-python/data-model.md`](../spec-python/data-model.md) §10 return the documented results after `php artisan migrate:fresh`; no column has a `DEFAULT` | pending |
| T-03 | Domain: `Isbn`, `Quantity`, `DateRange` (value objects) | T-01 | PHPUnit unit tests: ISBN of 13 digits, quantity > 0, end ≥ start | pending |
| T-04 | Domain: `Book` (`withdraw`, `restock`, soft delete) | T-03 | PHPUnit tests: never below 0, never above total | pending |
| T-05 | Domain: `Loan` (`addItem`, `returnItem`, calculated state, 14 days, max 3) | T-04 | PHPUnit tests with a fake `Clock`: `open`, `overdue`, `closed` | pending |
| T-06 | Domain: `Member` and `User` (name normalization, valid role) | T-03 | `"Ana "` becomes `ana`; role `admin` sent through the API is rejected | pending |
| T-07 | Ports and use cases for **authentication** (`LoginService`, `CreateLibrarianService`) with fakes | T-06 | HU-01 and HU-02 tests pass against in-memory fakes | pending |
| T-08 | Ports and use cases for **books** (create, search, edit, soft delete, cover) with fakes | T-04 | HU-03 and HU-04 tests pass against fakes | pending |
| T-09 | **Lend** use case (`LendBooksService`) with fakes | T-05, T-08 | HU-06 complete: limits, overdue, no copies | pending |
| T-10 | **Return** and **overdue** use cases | T-09 | HU-07 and HU-09 tests; returning twice → `ITEM_ALREADY_RETURNED` | pending |
| T-11 | **Most-borrowed report** use case and its read port (`ReportReader`) | T-09 | HU-10 test; the old title is kept after a rename (ADR-004) | pending |
| T-12 | Persistence adapter: book, member and user repositories (Query Builder) | T-02, T-08 | PHPUnit integration tests against real PostgreSQL 16 | pending |
| T-13 | Persistence adapter: loan repository + conditional update for copies | T-12, T-09 | Two loans for the last copy: one 201, one 409 (RNF-2). Test in `tests/Integration/` | pending |
| T-14 | Security adapter: password hashing (Argon2id) and JWT (`firebase/php-jwt`) | T-07 | The password never appears in logs or responses (RNF-3) | pending |
| T-15 | Cover storage adapter (local disk) | T-08 | Upload a 2 MB PNG: ok; 3 MB → 422; no cover → `cover_key` is `null` | pending |
| T-16 | HTTP adapter: routes of [`../spec-python/api-contract.md`](../spec-python/api-contract.md), single error body, role authorization | T-10, T-11, T-14, T-15 | Contract test (Feature) walks every route and every status code of §2 | pending |
| T-17 | Composition and boot (initial admin from environment) | T-16 | The app boots with an empty database and the admin can sign in through `POST /api/v1/auth/login` | pending |
| T-18 | Push the **domain-only** invariants down to the database: `ck_book_total_positive`, `ck_user_role` | T-12 | An `INSERT` through `psql` that breaks them fails; `data-model.md` §4 is updated | pending |
| T-19 | Architecture tests (Articles I, II and III) | T-17 | deptrac rule or PHPUnit scan in `tests/Architecture/` passes, and fails when a forbidden import is added | pending |
| T-20 | `infra`: `docker compose up` starts an empty database, the API and the frontend | T-17 | One command; the database has no tables until the migrations run | pending |
| T-21 | `tool`: demo seeder **through the API** (≥ 40 books, 10 members, 15 loans, 3 overdue) | T-17 | After running it, `GET /api/v1/loans/overdue` returns 3 lines and the report has rows | pending |
| T-22 | `app`: login, catalog, members, lend, return, overdue, report | T-16 | Manual end-to-end walkthrough in the browser (login → lend → return → report) | pending |
| T-23 | `page`: public static presentation site | — | Opens without the API; correct on phone and desktop. Source: `page/index.html`, `page/styles.css` | pending |

## Definition of done for the test

Whoever reaches T-17 with evidence has the **core** done. T-18 to T-23 are what separates good work from excellent work.
The work is assessed on the performance with SDD, not on the number of tasks: one task with evidence is worth more than three half-done ones.
