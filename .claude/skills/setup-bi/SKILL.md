---
name: setup-bi
description: Auto-detect a database file in this folder and interactively set up Metabase, Redash, or Superset connected to it.
---

# BI Tool Setup

## Step 1: Find the data file automatically
Look in the current working directory (the same folder as this skill's
project) for a database file: a `.sql` dump, a `.csv` file, or a `.db`
(SQLite) file.

- If exactly one candidate file is found, use it — no need to ask which
  file, just confirm to the user which one you found before proceeding.
- If more than one is found, list them and ask the user which one to use.
- If none are found, tell the user plainly and ask them to point you to
  the file or folder.

## Step 2: ALWAYS ask which tool — never assume, never proceed silently
Once the data file is identified, stop and explicitly ask the user:

> "Found `<filename>`. Which BI tool would you like to set up:
> Metabase, Redash, or Superset?"

Wait for their actual answer before doing anything else. Do not guess
based on prior conversation history, do not default to whichever tool was
used last time, and do not proceed on an assumption. This is a required
stop-and-ask point, not optional.

## Step 3: Follow the matching section of db-to-bi-tool-skill.md
Once the user has answered, read `db-to-bi-tool-skill.md` (expected to be
in the same folder) and follow the section matching their choice
(Metabase / Redash / Superset) exactly, including all tool-specific steps
(e.g. Redash's `.env` secrets and `create_db` step, Superset's init step).

## Step 4: After setup completes, ask — don't pre-fill or assume the next action
Once the chosen tool is up and verified, do not automatically suggest,
type, or pre-fill a follow-up command (such as "add the database
connection for me") into the next prompt. Instead, explicitly ask the
user an open question and wait for their real response, for example:

> "Setup is complete and verified. Would you like me to also add the
> target database connection inside <tool>, or would you prefer to do
> that yourself through the UI?"

The user should always be the one deciding and typing the next action —
your job at this point is to report status and ask, not to assume what
they want done next.

## Note on this behavior
Claude Code's interface may sometimes suggest a follow-up prompt on its
own as part of normal product behavior — that's outside this skill's
control. But within the instructions this skill gives you, the rule is
firm: always phrase next steps as an explicit question and wait for the
user's typed response, rather than treating a suggested action as already
decided.
