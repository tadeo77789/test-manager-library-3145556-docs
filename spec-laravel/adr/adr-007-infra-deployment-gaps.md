# ADR-007 · Infrastructure and deployment gaps (not fixed)

- **Status:** accepted — known deviation, not fixed (13-oct-2026).
- **Context:** `infra/docker-compose.yml` brings up `db`, `api` and `app` with one command and an empty database (T-20, verified).
  The review found gaps that do not break the demo but matter outside it:
  1. **Covers path:** `infra/.env.example` sets `COVERS_DIR=/var/www/html/storage/covers`, while the `api` image uses `WORKDIR /var/www`.
     The API falls back to `storage/covers` when `COVERS_DIR` is unset or not writable, so it works, but the two repositories disagree.
  2. **Image tag:** the `db` service uses `postgres:16`; development used `postgres:16-alpine`. Both are PostgreSQL 16 (D-03), but the
     first run downloads a second image.
  3. **Optional `.env`:** `env_file` is marked `required: false`. Without `cp .env.example .env` the API starts without `JWT_SECRET`
     and without `ADMIN_USERNAME`/`ADMIN_PASSWORD`, so nobody can sign in.
  4. **No volume for covers:** cover binaries live inside the `api` container; recreating it loses them while `cover_key` stays in the
     database (`GET /books/{id}/cover` → 404 for those books).
- **Decision:** keep `infra` as delivered. The required step is: **copy `.env.example` to `.env` and set real secrets before
  `docker compose up --build`** (Article IX: secrets come from the environment).
- **Discarded alternatives:** editing `infra` during the closing window (closing rule); making `.env` mandatory now would also change
  the documented one-command flow.
- **Consequences:** the demo works with the documented steps. For a durable deployment: a named volume mounted on the covers directory,
  one agreed `COVERS_DIR`, `env_file` required, and a single PostgreSQL 16 tag. ADR-001 is not affected: `infra` still has no DDL.
- **How to verify:** `grep -n "COVERS_DIR" ../infra/.env.example`; `grep -n "WORKDIR" ../api/Dockerfile`;
  `grep -n "required: false" ../infra/docker-compose.yml`; `grep -n "volumes" -A3 ../infra/docker-compose.yml` shows only `db_data`.
