# Technical plan — Manager Library (PHP 8.3 / Laravel + React)

Translation of [`../spec-python/plan.md`](../spec-python/plan.md). The business rules, the HTTP contract and the data model do not change.
How the system is built does. The data model is not described here: it lives in [`../spec-python/data-model.md`](../spec-python/data-model.md).
The HTTP contract lives in [`../spec-python/api-contract.md`](../spec-python/api-contract.md). The system shape is in [`architecture.md`](architecture.md).

## 1. Technical decisions

| # | Decision | Choice (Laravel / React) | Why |
|---|---|---|---|
| D-01 | Language and runtime | PHP 8.3 (`readonly` classes, enums, `match`, first-class callables) on the Laravel 11 framework; frontend on Node 24 | The challenge stack. PHP 8.3 is the oldest version with all the features the domain uses |
| D-02 | HTTP framework | Laravel routing (`routes/api.php`, prefix `/api/v1`), form requests and a single exception handler | The contract in [`api-contract.md`](../spec-python/api-contract.md) maps directly to routes |
| D-03 | Persistence | Laravel migrations, with **Query Builder** (`DB::table('library.book')`) in the persistence adapters on PostgreSQL 16. Eloquent is not used in the domain or in the application layer | The schema is owned by the migrations ([ADR-001](../spec-python/adr/adr-001-propiedad-del-esquema.md)). Query Builder keeps the adapters explicit and the SQL visible |
| D-04 | Concurrency | Conditional update `UPDATE library.book SET available_copies = available_copies - 1 WHERE id = ? AND available_copies = ?` plus a `CHECK` range, instead of a version column ([ADR-002](../spec-python/adr/adr-002-concurrencia-optimista.md)). See [`translation.md`](translation.md) (c) | Last copy (RNF-2). The data model has no `version` column |
| D-05 | Dates | UTC instants (`timestamptz`); the clock is a port (`Clock`) with an adapter that reads the system clock, plus an optional offset for demo data (see `CLOCK_OFFSET_DAYS`) | Test overdue loans without waiting 14 days |
| D-06 | Report | One aggregated query, behind a read port (`ReportReader`), see [ADR-004](../spec-python/adr/adr-004-reporte-agregado-y-congelado.md) | Stable and cheap |
| D-07 | Value objects | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` are `readonly` classes inside `App\Domain`, stored inside the row of their owner, with no table | They have no identity |
| D-08 | Covers | Binary in external storage (`CoverStorage` port, local disk in development, S3 in production via the `Storage` facade adapter); the database only stores `cover_key` | Opaque key, never a path |
| D-09 | Passwords | Argon2id through the `PasswordHasher` port, implemented with PHP's native `password_hash($plain, PASSWORD_ARGON2ID)` and `password_verify` | Article IX. The domain never sees the clear password |
| D-10 | Bootstrap | The application creates the initial admin from `ADMIN_USERNAME` / `ADMIN_PASSWORD` when it boots, if it does not exist | The database cannot produce the hash |
| D-11 | Authentication | JWT HS256 with `firebase/php-jwt`, 8 hours (`exp = now + 28800`), key in `JWT_SECRET`. The `TokenIssuer` port hides the library | Simple and stateless |
| D-12 | Tests | PHPUnit 11. Fakes for the ports in the application tests; a real PostgreSQL 16 only in integration and concurrency tests. Architecture tests with deptrac or a PHPUnit scan | Test pyramid |

Frontend (`app`): React with Vite, React Router, plain `fetch` to the API through one small client module. The UI texts stay in Spanish.

## 2. Environment configuration

| Variable | Required | Example |
|---|---|---|
| `DB_CONNECTION` | yes | `pgsql` |
| `DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | yes (or `DATABASE_URL`) | `db` / `5432` / `library` / `library` / *(in `.env`)* |
| `DATABASE_URL` | optional alternative to the `DB_*` group | `postgresql://library:library@db:5432/library` |
| `DB_SCHEMA` | no | `library` |
| `JWT_SECRET` | yes | long random string |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | yes | defined in `.env`, never in the repository |
| `COVERS_DIR` | no | `storage/app/covers` |
| `CORS_ORIGIN` | yes | `http://localhost:5173` |
| `CLOCK_OFFSET_DAYS` | no, default `0` | `-15` (see [`translation.md`](translation.md) (a)). Read only by the `Clock` adapter |
| `APP_KEY` | yes (Laravel) | generated with `php artisan key:generate`, never committed |

## 3. Test strategy

| Level | What it tests | Against what |
|---|---|---|
| Domain | Invariants: maximum 3 books, 14-day due date, available copies within range | Nothing: pure objects with PHPUnit |
| Application | Use cases: lend, return, report | Fakes of the ports |
| Integration | Repositories and the report query | Real PostgreSQL 16 |
| Contract | Single error body and status codes | The API running (`php artisan serve` or the container) |
| Architecture | Articles I, II and III | deptrac rules or a PHPUnit scan in `tests/Architecture/` |
| Concurrency | Two loans for the last copy | Real PostgreSQL 16 |

## 4. Build order

Skeleton and health → initial migration with seed → domain → use cases with fakes → adapters → HTTP routes → contract and concurrency tests.
Details and evidence are in [`tasks.md`](tasks.md).

## 5. Docker constraint

PHP and Composer are not installed on the development host. They run in containers:
`docker run --rm -v "$PWD":/app -w /app composer:2 install`, and PHP 8.3 CLI with the `pdo_pgsql` extension for `php artisan` and `phpunit`.
