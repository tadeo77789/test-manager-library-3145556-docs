# Constitution — Manager Library (PHP 8.3 / Laravel + React)

Non-negotiable principles of the project, translated from [`../spec-python/constitution.md`](../spec-python/constitution.md).
The rules are the same; only the tools that check them change. Each article states **how it is verified** in this stack,
because a principle that nobody can check is only an intention. A change that breaks an article is rejected, whatever else it does.

Language: this document is in English (Article XI). Error messages of the API and the user interface stay in Spanish.

---

## Article I — The domain depends on nothing

`app/Domain/` is plain PHP 8.3. It does not import `Illuminate\*`, Eloquent, facades, helpers that come from a package,
or any third-party library. Value objects and entities are `final` classes with `readonly` properties (`readonly` classes or `readonly` promoted properties).

**Why:** it is the only way to test the invariants without a database, a server or a container, and to change the framework without touching the rules.
**Verification:** a deptrac rule (or a PHPUnit architecture test under `tests/Architecture/` that scans the namespace) forbids `App\Domain` from depending on
`Illuminate`, `App\Application`, `App\Adapters` or `App\Bootstrap`. Any `use Illuminate\…` inside `app/Domain/` fails the build.

## Article II — The core declares its ports; the outside implements them

`app/Application/` only knows `App\Domain`. Everything the use cases need from the outside (repositories, clock, password hashing, token issuing,
cover storage, report reading) is an **interface that Application defines itself**, under `app/Application/Ports/`. No framework or ORM type
appears in a port signature: no `Request`, no `Model`, no `Builder`, no `Carbon`, no `Collection`.

**Verification:** deptrac forbids `App\Application` from depending on `App\Adapters` and `Illuminate`, except in the service provider that is
explicitly excluded (see Article III). The application tests run with fakes (in-memory classes implementing the ports) and never reference `App\Adapters`.

## Article III — A single composition point

Only one class, `app/Bootstrap/PortBindingsServiceProvider.php`, binds a port to its adapter (`$this->app->bind(LoanRepository::class, EloquentLoanRepository::class)`
and so on). Outside it, no adapter is instantiated with `new` and no adapter is resolved by its concrete class.

**Verification:** an architecture test greps `app/` for `new` of a class in `App\Adapters` outside the provider, and for `app(…Adapter…)` calls.
Laravel's container is used only inside the provider; controllers receive ports through constructor injection.

## Article IV — SOLID is not decoration

| Principle | Concrete application | Verification |
|---|---|---|
| **SRP** | One application service = one use case. `LendBooksService` lends and does nothing else | A service with two reasons to change is split |
| **OCP** | Changing cover storage (local disk → S3) does not touch Domain or Application | The new adapter only adds one binding in the provider |
| **LSP** | Test fakes replace the real adapters without changing the expectations | The same contract test passes against the fake and against the Eloquent / filesystem adapter |
| **ISP** | Small ports: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | No implementer leaves methods unused or throws "not supported" |
| **DIP** | The port is defined by the consumer | Ports live in `App\Application\Ports`, not in `App\Adapters` |

## Article V — The schema belongs to the migrations

The `infra` repository starts an **empty** PostgreSQL 16 server. All DDL (tables, indexes, constraints, extensions and seed data) lives in
**Laravel migrations** in the `api` repository, under `database/migrations/`. See [`../spec-python/adr/adr-001-propiedad-del-esquema.md`](../spec-python/adr/adr-001-propiedad-del-esquema.md).
The `library` schema is created by a migration. Laravel's default tables (`users`, `password_reset_tokens`, `jobs`, …) are not used by the domain.

**Verification:** `grep -ri "create table" infra/` returns nothing. `php artisan migrate:fresh` on an empty database produces exactly the tables of
[`../spec-python/data-model.md`](../spec-python/data-model.md).

## Article VI — The invariants the database can enforce, the database enforces

A rule that only lives in PHP protects the application, not the data. Every invariant that PostgreSQL can express (`CHECK`, `UNIQUE`, foreign keys,
partial unique indexes) is declared in a migration. What is not enforced yet is marked **pending** in `data-model.md`; it is not promised.

**Verification:** each constraint of [`../spec-python/data-model.md`](../spec-python/data-model.md) §4 and §10 exists in the migrations. A PHPUnit integration test
inserts a row that breaks each `CHECK` through the query builder and expects a `QueryException` with the constraint name.

## Article VII — Derived values are calculated, not stored

The state of a loan (`open`, `overdue`, `closed`) and the days late are computed by the domain (`App\Domain\Loan`) from `due_at` / `returned_at` and the
`Clock` port. There is no column for them.

**Verification:** no migration contains a `status` or `days_late` column (checked by the same test that reads `data-model.md`). Domain tests use a fake clock.

## Article VIII — Frozen values do not follow the catalog

The loan line stores the title, ISBN and category at the moment of the loan. No report reads them from the live catalog.

**Verification:** an integration test renames a book after lending it and checks that the most-borrowed report still shows the old title
([`../spec-python/adr/adr-004-reporte-agregado-y-congelado.md`](../spec-python/adr/adr-004-reporte-agregado-y-congelado.md)).

## Article IX — No secret enters the repository or a log

No passwords, no precomputed hashes, no token key. Every secret comes from the environment (`.env`, read by Laravel's `config/`).
The real `.env` is in `.gitignore`; only `.env.example` is versioned. `APP_KEY` is generated on the machine, never committed.
Logs never contain the password or the `Authorization` header.

**Verification:** `gitleaks` or a `grep` for key patterns in CI; no HTTP response contains `password` or `password_hash`
(a PHPUnit contract test asserts this for `POST /users` and `POST /auth/login`).

## Article X — The engine wins

If the data document and the database disagree, the document is broken and the document is corrected.

**Verification:** when a migration and `data-model.md` differ, the migration is the reference only if the document is updated in the same change.
Migrations are the executable form of the data model.

## Article XI — Language

Code, columns, routes, commits and documentation are in **English**. This is a decision of the project owner for this translation.
The `message` field of API errors and the text shown to the librarian in the interface stay in **Spanish**, because the HTTP contract requires it
([`../spec-python/api-contract.md`](../spec-python/api-contract.md) §1).

**Verification:** `grep` for Spanish identifiers in `app/` and `src/` returns nothing; the error bodies of the contract tests contain the Spanish messages of the contract.

## Article XII — Nothing is considered done without evidence

A task of [`tasks.md`](tasks.md) is finished when its **evidence** exists and can be executed: a PHPUnit test, an `artisan` command, a `curl` call or a
browser walkthrough. "It is coded" is not evidence. Reports list each task as T-xx with the command or test that proves it.
