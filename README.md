# DB-to-BI-Tool Setup Agent

A Claude Code skill that automates standing up a local BI/analytics
environment (Metabase, Redash, or Apache Superset) connected to your own
data — starting from just a folder containing a database file.

## What it does

Run one command, `/setup-bi`, inside a folder containing a data file:

1. **Auto-detects** the data file (`.sql` dump, `.csv`, or `.db` SQLite file)
in the folder
2. **Asks which BI tool** you want — Metabase, Redash, or Superset — rather
than assuming
3. **Builds and starts** the correct Docker Compose stack for that specific
tool, handling each one's real setup quirks (see below)
4. **Verifies** the setup actually works, not just that containers started
5. **Asks before doing anything further** — e.g. whether to open the tool
in your browser — rather than assuming what you want next

## Why this isn't just "swap the Docker image"

Metabase, Redash, and Superset have genuinely different setup requirements,
not just different container names:

|                                        | Metabase                   | Redash                                                                         | Superset                          |
| -------------------------------------- | -------------------------- | ------------------------------------------------------------------------------ | --------------------------------- |
| Needs its own metadata DB?             | No (bundled H2 by default) | Yes, separate Postgres                                                         | Yes, separate Postgres            |
| Needs Redis?                           | No                         | Yes                                                                            | Yes                               |
| One-time init step before first start? | No                         | Yes — `create_db`                                                              | Yes — admin/schema init           |
| Requires generated secrets?            | No                         | Yes — `REDASH_SECRET_KEY`, `REDASH_COOKIE_SECRET` (per Redash's official docs) | Yes — via init script             |
| Connects to your data during setup?    | Yes (setup wizard)         | No — added manually/via API after                                              | No — added manually/via API after |

Getting these wrong produces stacks that either fail to start, or start but
silently use insecure defaults. This project's `db-to-bi-tool-skill.md` encodes the correct sequence for each tool, verified against each tool's
official documentation and confirmed by an actual working end-to-end run.

## Verification

All three tools were tested against the same synthetic dataset (300 users,
~780 orders, ~430 support tickets) and each independently returned the same
ground-truth query result:
