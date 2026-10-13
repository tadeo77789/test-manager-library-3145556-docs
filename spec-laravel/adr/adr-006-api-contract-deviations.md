# ADR-006 · API behaviour not defined, or defined differently, by the contract (not fixed)

- **Status:** accepted — known deviation, not fixed (13-oct-2026).
- **Context:** the `api` repository implements every route of [`api-contract.md`](../../spec-python/api-contract.md). Where the
  contract is silent, the implementation took decisions; in a few places it goes beyond the contract. They were found after the
  core was delivered and, under the closing rule, were recorded instead of changed:
  1. **Extra error codes** not listed in contract §2: `UNAUTHENTICATED` (401, missing/invalid/expired token), `METHOD_NOT_ALLOWED` (405)
     and `INTERNAL_ERROR` (500). The body shape (`code`, `message`, `details`) is the contract's single error body.
  2. **Password minimum of 8 characters** when creating a librarian. The spec does not define a password policy.
  3. **`size` greater than 50 returns 422 `VALIDATION_ERROR`** instead of being clamped to 50. The contract only says "maximum 50".
  4. **`GET /loans/overdue` is paginated** (`{items, page, size, total}`, default size 50). The contract shows `items` without saying
     whether the list pages.
  5. **Server process:** the container runs `php artisan serve` with 4 workers (needed so two requests can run at the same time for RNF-2).
     It is a development server, not a production one.
  6. **CORS preflight:** a preflight from a disallowed origin still answers 204 with `Access-Control-Allow-Origin: <CORS_ORIGIN>`
     (static value). Browsers block the request correctly because the origin does not match, so it is not exploitable, but the
     response is not a refusal.
- **Decision:** keep the behaviour for the delivery and declare it here. 1 (401 for a bad token) and 4 are the interpretations the
  team considers correct; 2, 3, 5 and 6 are accepted limitations.
- **Discarded alternatives:** changing the code during the closing window (closing rule); changing `api-contract.md`
  (the contract is shared with the Python and .NET versions and must not be edited from this translation).
- **Consequences:** a client written only from the contract must also handle `UNAUTHENTICATED`, `METHOD_NOT_ALLOWED` and `INTERNAL_ERROR`,
  must not send `size > 50`, and must read `/loans/overdue` as a paginated list. The `app` and `tool` repositories already do.
  Production would need a real PHP server (php-fpm + nginx, or Octane) and a CORS layer that refuses foreign origins.
- **How to verify:** `curl -i localhost:8000/api/v1/books` without token → 401 `UNAUTHENTICATED`;
  `curl -i "localhost:8000/api/v1/books?size=51" -H "Authorization: Bearer …"` → 422;
  `curl -i -X OPTIONS localhost:8000/api/v1/books -H "Origin: http://evil.example" -H "Access-Control-Request-Method: GET"` → 204 with the static origin.
