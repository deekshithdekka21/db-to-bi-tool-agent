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

To prove the setup actually works (not just that containers start), all three
tools were tested against the same synthetic dataset during development, and
each independently returned matching results for the same query run against
their own connection to that data.

If you're using this with your own data file, you can verify your setup the
same way: run any query you already know the answer to (e.g. a row count or
a `GROUP BY` on a column you recognize) directly in the BI tool's query
interface, and confirm it matches what you'd expect from the source file
itself. If you test with more than one tool, running the identical query in
each and confirming they return the same result is a good sanity check that
every tool is actually connected to the same underlying data, not just that
its UI loaded successfully.

## Real debugging along the way

- Caught a missing requirement in the original setup instructions:
self-managed Redash deployments need `REDASH_SECRET_KEY` and
`REDASH_COOKIE_SECRET` explicitly generated, per Redash's official docs —
this wasn't obvious from community setup guides alone.
- Fixed a case where the agent's first attempt at adding a Redash data
source failed because a startup script mis-parsed a data source name
containing spaces; it diagnosed the cause and retried using Redash's CLI
tool directly.
- Verified container provenance directly (which `docker-compose.yml` each
running container actually started from) after a leftover file from an
earlier test was found sitting in a test folder, to confirm test results
weren't accidentally validating stale infrastructure.

## Files

- **`db-to-bi-tool-skill.md`** — the core instructions: how to detect a
data file, and the exact setup sequence for each of the three tools
- **`.claude/skills/setup-bi/SKILL.md`** — a Claude Code Skill that wraps
the above into a single `/setup-bi` command, with interactive prompts
instead of a long typed instruction each time

## Setup

Before running `/setup-bi`, you'll need Claude Code installed and connected to your Claude account:

1. Install the **Claude Code** extension in VS Code (or use the Claude Code CLI directly in your terminal).
2. In your terminal, navigate to the folder containing your data file (`.sql`, `.csv`, or `.db`), e.g. `cd path/to/your/folder`.
3. From inside that folder, run `claude` to authenticate — this connects your Claude account and opens a Claude Code session with that folder as its working directory.
4. Run `/setup-bi` inside that Claude Code session.

## Usage

```
# In a folder containing your .sql / .csv / .db file:
claude
> /setup-bi
```

It will find your data file, ask which tool you want, and take it from
there.
