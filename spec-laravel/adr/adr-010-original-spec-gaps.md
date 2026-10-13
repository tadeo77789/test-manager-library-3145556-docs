# ADR-010 · Gaps and inconsistencies in the original spec

- **Status:** accepted (13-oct-2026). Recorded, not changed: `spec-python/` and `spec-.net/` belong to the authors of the test and were not edited.
- **Context:** while translating the spec to Laravel + React, these points of the **original** spec (identical in the Python and .NET
  versions) turned out to be missing, contradictory or silent. Each one forced a decision in the translation:
  1. **ADR-002 vs data model:** ADR-002 asks for a *version token* per book, but `data-model.md` §3 has no `version` column and
     forbids columns not listed there. → The translation detects the last-copy race with a conditional
     `UPDATE … SET available_copies = available_copies - 1 WHERE id = ? AND available_copies = ?` plus `ck_book_available_range`.
  2. **Seeding 3 overdue loans "through the API"** (T-21) while `loaned_at` is set by the server clock (CA-06.6, D-05).
     No endpoint accepts a past date. → The `Clock` adapter reads an optional `CLOCK_OFFSET_DAYS` (default 0); the seeder runs once with `-20`.
  3. **`GET /books/{id}/cover`** is required by the text of `api-contract.md` §3.4 ("to paint the cover use GET /books/{id}/cover")
     but is missing from the endpoint table, so its role is not stated. → Implemented for any authenticated user, returns the binary or 404.
  4. **Route overlap:** `GET /loans/overdue` and `GET /loans/{id}` share a prefix; the contract does not say which wins.
     → `/loans/overdue` is registered first.
  5. **401 without a code:** §2 defines `INVALID_CREDENTIALS` for login only; it gives no `code` for a missing, invalid or expired token,
     although every error must use the single body. → `UNAUTHENTICATED` (see ADR-006).
  6. **Silent points:** password policy (none given → minimum 8 characters), behaviour of `size > 50` (clamp or reject → reject with 422),
     whether `GET /loans/overdue` is paginated (→ paginated like every list).
  7. **Concurrency only for the last copy:** RNF-2 and ADR-002 cover two loans of the last copy, but RN-2/RN-3 (max 3 unreturned,
     no loans with overdue items) are marked "domain only" with no concurrency requirement. Two simultaneous loans for the same member
     are not covered by the spec. → The translation re-checks both rules inside the loan transaction with a lock on the member row
     when that fix is delivered (see the `api` README); otherwise it is a known limitation.
- **Decision:** keep the original documents untouched and apply the choices above in `spec-laravel/` and the code.
- **Discarded alternatives:** editing `spec-python/` or `spec-.net/` (they are the shared source of truth of the test); adding a
  `version` column (would contradict `data-model.md` and Article X).
- **Consequences:** a reviewer comparing the code with the original spec will find these differences explained here and in
  [`translation.md`](../translation.md). The spec authors may want to close points 1, 3, 5 and 7 in a future revision.
- **How to verify:** `grep -n "version" ../../spec-python/adr/adr-002-concurrencia-optimista.md ../../spec-python/data-model.md`;
  `grep -n "cover" ../../spec-python/api-contract.md`; `grep -n "INVALID_CREDENTIALS" ../../spec-python/api-contract.md`.
