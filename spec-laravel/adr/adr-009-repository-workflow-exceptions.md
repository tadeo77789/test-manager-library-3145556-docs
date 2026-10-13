# ADR-009 · Repository workflow and its exceptions

- **Status:** accepted — exceptions recorded, history not rewritten (13-oct-2026).
- **Context:** the team agreed these working rules during the test:
  - branches `feature/*`, `docs/*` and `fix/*` are created from `develop`, merged into `develop` with `--no-ff`, and `develop` is
    promoted to `main` only when a block is complete (`main` is what is delivered);
  - commits and documentation in English, except the API `message` field and UI texts, which stay in Spanish (Article XI, contract);
  - every commit and merge message ends with the line `Note: working over mobile (cellular) internet connection.`;
  - pinned dependencies with lockfiles; `.env`, `vendor/` and `node_modules/` are never committed.
  The rules arrived progressively, so some earlier commits do not follow them:
  1. Commits made directly on `main` before the branch rule existed: `app` (`c696ad4`), `page`, `docs` (the first `spec-laravel/`
     documents), `infra` and `tool` (first T-20/T-21 commits).
  2. The first three merges of `api` into `develop` (T-01, T-02, T-03..T-06) lack the mobile-internet line.
  3. `app/Dockerfile` line 1 has a comment in Spanish.
- **Decision:** keep the history as it is. Rewriting published history (`rebase`, `push --force`) is forbidden by the same rules and
  would make the delivery harder to audit. From the moment each rule was agreed, every commit follows it.
- **Discarded alternatives:** amending or rebasing the old commits; deleting and re-creating branches.
- **Consequences:** `git log` shows a few direct commits on `main` at the start of each repository and some messages without the note.
  Everything delivered after the rules were set goes through `develop`.
- **How to verify:** `git log --first-parent --oneline origin/main` in each repository (merge commits from `develop` after the rules);
  `git log --format=%B origin/develop | grep -c "Note: working over mobile"`; `sed -n 1p ../app/Dockerfile`.
