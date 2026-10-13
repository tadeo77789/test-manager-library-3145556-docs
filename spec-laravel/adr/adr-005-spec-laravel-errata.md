# ADR-005 · Errata of the `spec-laravel/` documents (not fixed)

- **Status:** accepted — known deviation, not fixed (13-oct-2026).
- **Context:** a review of `spec-laravel/` against the delivered code and the other repositories found statements that
  are wrong or inconsistent. The closing rule of the test was "record the fault, do not fix it", so the documents were not edited.
  The faults:
  1. `plan.md` (env table, `CLOCK_OFFSET_DAYS`) gives `-15` as the example; `tool/README.md` and `infra/.env.example` use `-20`.
     With `-15` a loan is overdue by only 1 day after the clock is reset, which is fragile.
  2. `plan.md` D-01 says **Laravel 11**; `api/composer.json` requires `laravel/framework ^13.17`.
  3. `constitution.md` (Article III) uses `EloquentLoanRepository` as its example, while `plan.md` D-03 decides **Query Builder, no Eloquent**.
     The delivered adapters are `PgBookRepository`, `PgLoanRepository`, … (Query Builder).
  4. `architecture.md` (lend sequence) says "ok, or version conflict"; decision (c) of `translation.md` is a conditional
     `UPDATE … WHERE available_copies = ?` with no `version` column.
  5. `translation.md` says the .NET version "was not used as a line-by-line cross-check". It was compared file by file:
     `spec.md`, `data-model.md`, `api-contract.md` and the four ADRs are identical; only `constitution.md`, `plan.md`,
     `architecture.md`, `arquitectura-panoramica.md` and `tasks.md` differ, and only in technology.
  6. `translation.md` decision (f) says the project brief names `spec-python/` as the reference version. The brief does not choose
     either version; `spec-python/` was chosen by the team because the business is identical in both.
- **Decision:** this ADR is the correction. Where a `spec-laravel/` document and this ADR disagree, **this ADR wins**:
  1. The demo offset is `CLOCK_OFFSET_DAYS=-20`.
  2. The framework is Laravel 13 (`^13.17`) on PHP 8.3.
  3. Persistence adapters use Query Builder; there are no Eloquent repositories.
  4. The last-copy race is detected by the conditional `UPDATE` (affected rows = 0 → `BOOK_NOT_AVAILABLE`), not by a version token.
  5. Both spec versions were compared; the business is identical.
  6. The choice of `spec-python/` as the base is a team decision, not a requirement of the brief.
- **Discarded alternatives:** editing the documents in place (forbidden by the closing rule and it would hide that the fault existed).
- **Consequences:** a reader of `spec-laravel/` must read this ADR together with it. Article X of the constitution still applies:
  if a document and the running system disagree, the document is wrong.
- **How to verify:** `grep -n "Laravel 11" spec-laravel/plan.md`; `grep -n "^\s*\"laravel/framework\"" ../api/composer.json`;
  `grep -rn "Eloquent" ../api/app/Adapters/Persistence` returns nothing; `grep -n "CLOCK_OFFSET_DAYS" ../tool/README.md ../infra/.env.example`.
