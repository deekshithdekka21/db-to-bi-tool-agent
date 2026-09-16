# DB-to-BI-Tool Setup Agent

## Purpose
Given a folder containing one or more database files, stand up a local
Docker Compose stack loaded with that data, connected to a BI tool of the
user's choice (Metabase, Redash, or Apache Superset), and report back the
exact connection/login details needed to start using it.

## Step 0: Ask which tool, if not already specified
Don't assume. Ask the user which BI tool they want: Metabase, Redash, or
Superset. Each has meaningfully different setup requirements (see below) —
guessing wrong wastes a full setup cycle.

## Step 1: Handle the data file (same for all three tools)
Same logic regardless of which BI tool is chosen:

- **`.sql` file**: assume it's a Postgres dump, mount it as a Postgres
  init script so it loads automatically on first container start.
- **`.csv` file(s)**: infer column types from the header/sample rows,
  generate a `CREATE TABLE` + load script, and run it via `docker cp` +
  `psql \copy` after the container starts (init scripts can't reach a file
  that isn't already inside the container).
- **`.db` file (SQLite)**: ask whether the user actually wants a full
  Docker/BI-tool setup, or whether DB Browser for SQLite alone is enough —
  don't assume. If they want the full stack, convert it to a Postgres dump
  first, then proceed as with a `.sql` file.

This target data always lives in its own Postgres container/database,
separate from whatever internal metadata database the BI tool itself needs
(see below) — never mix the two.

## Step 2: Tool-specific setup

### Metabase (simplest)
- One `metabase` service (image `metabase/metabase:latest`), one
  `postgres` service holding the target data.
- Don't set `MB_DB_*` env vars — Metabase defaults to its own bundled H2
  database for its internal app metadata, keeping it cleanly separate from
  the data being explored.
- No special init command needed. `docker compose up -d` is sufficient.
- First visit to `localhost:3000` shows a setup wizard (admin account +
  "Add your data" step) if this is a fresh Metabase instance, or a login
  screen if an admin account already exists from a previous run.

### Redash (moderate complexity)
- Needs FOUR things running: `server`, `scheduler`, `worker` (all from the
  `redash/redash` image), plus its own `postgres` and `redis` for internal
  metadata/job queueing — this is separate from the target-data Postgres
  container from Step 1.
- **Before first startup**: create a `.env` file (never commit this) with
  two generated secret values Redash requires for a self-managed Docker
  setup: `REDASH_SECRET_KEY` and `REDASH_COOKIE_SECRET`. Generate each with
  something like `pwgen -1s 32` or `openssl rand -hex 32` — don't leave
  these unset or reuse a placeholder, since they're used for session
  cookies and encryption.
- **Critical, easy-to-miss step**: before the first `docker compose up -d`,
  you must run `docker compose run --rm server create_db` once to
  initialize Redash's own internal database schema. Skipping this causes
  the server to fail on startup with a database error.
- After it's running, visit `localhost:5000` (or whatever port is mapped).
  Redash's own first-visit setup screen will prompt you to create the
  initial admin account directly in the browser — this happens
  automatically on first load, not via a separate CLI step. Once done, add
  the target Postgres data source manually via Redash's UI (Settings →
  Data Sources) — Redash doesn't have an equivalent of Metabase's
  during-setup "Add your data" step.

### Apache Superset (most complex)
- The official Superset repo's `docker-compose.yml` is heavy (nginx, redis,
  a `superset-init` one-shot service, `superset-worker`, `superset-node`,
  `superset-websocket`, plus its own postgres) — mainly intended for
  Superset's own development, not for quickly trying it out.
- For a quick local trial, prefer a minimal community-style compose file:
  one `superset` service, one `postgres` for its internal metadata, one
  `redis` for caching — not the full official multi-service setup.
- Requires a one-time init step (commonly a `setup.sh` or
  `docker compose exec superset superset-init`) to create the internal
  database schema and an admin account before first login.
- After startup, visit the mapped port (commonly `8088`), log in with the
  admin account created during init, then add the target Postgres
  connection manually via Superset's UI (Settings → Database Connections).

## Step 3: Verify and report back
1. Confirm all containers report healthy/running via `docker compose ps`.
2. If a CSV load step was needed, verify the row count matches the
   original file.
3. Report clearly to the user:
   - Which tool was set up and at what local URL.
   - The target-data Postgres connection details (host, port, db name,
     user, password) needed to add it inside the BI tool's UI, if the tool
     doesn't auto-connect it during its own setup wizard.
   - Any tool-specific first-login step they need to complete manually
     (e.g., Redash and Superset both require creating a source connection
     after the fact; Metabase may prompt for it during setup, or may
     already have an existing admin account from a prior run).
   - The relevant `docker compose down` / `down -v` cleanup commands.

## Things to double-check before finishing
- Never mix the BI tool's own internal metadata database with the target
  data being analyzed — always separate containers/databases.
- Redash's `create_db` step and Superset's init step are easy to forget
  because Metabase doesn't need an equivalent — don't assume all three
  tools behave the same way just because the first one worked cleanly.
- If the BI tool already has an existing admin account from a previous run
  (rather than being a fresh install), say so plainly rather than guessing
  whether login will work — you can't verify a password-protected UI on
  the user's behalf.
