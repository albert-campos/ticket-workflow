# ticket-workflow

Three Claude Code skills that keep ticket work honest across context resets.

| Skill | What it does |
|---|---|
| `intake` | Fetches the Jira ticket (description, acceptance criteria, **full comment thread**), checks local database freshness, and prints an approval digest. Nothing in the working tree changes until you approve it. |
| `offload` | Writes active session state to `PLAN.md` and `TODO.md` before `/clear` or a context reset. Records decisions and dead ends, not a conversation summary. |
| `resume` | Re-hydrates from `PLAN.md` and `TODO.md` in a fresh session, reports where work stands, and stops for your confirmation. |

The problem it solves: comments on a ticket routinely carry scope changes that never make
it into the description, and a context reset loses the reasoning behind decisions already
made. `intake` surfaces the first; `offload`/`resume` survive the second.

## Install

```
/plugin marketplace add albert-campos/ticket-workflow
/plugin install ticket-workflow
```

Then restart Claude Code so the hooks register.

## Configuration

Set these in `~/.claude/settings.json` (all machines) or a repo's `.claude/settings.json`
(that project only). Every one has a working default.

```json
{
  "env": {
    "TICKET_PREFIX": "ISI",
    "DB_BACKUP_DIR": "/c/Users/Public/Downloads",
    "SQL_SERVER": "localhost",
    "TARGET_DB": "MyAppDb"
  }
}
```

| Variable | Default | Purpose |
|---|---|---|
| `TICKET_PREFIX` | `ISI` | Your Jira project key. Drives both skill matching and the hooks. |
| `DB_BACKUP_DIR` | `/c/Users/Public/Downloads` | Where dropped `.bak` files land. |
| `SQL_SERVER` | `localhost` | `sqlcmd -S` target. |
| `TARGET_DB` | unset | Database a `.bak` restores into. Unset means the skill asks. |

## Requirements

- **Atlassian MCP server** connected — `intake` calls `getJiraIssue`. Without it the
  ticket fetch fails and the skill stops rather than guessing.
- **`sqlcmd` on PATH** — optional. Missing means the database section reports `Unknown`;
  the rest of the digest still works.
- **Git Bash or any POSIX shell** — the hooks run under `bash`.

## How the gate fires

Two hooks ship with the plugin:

- `UserPromptSubmit` — a ticket ID typed in a prompt (`ISI-4809`, `isi_4809`, `isi4809`)
  injects a reminder to run `intake` first.
- `SessionStart` — a branch name containing a ticket ID does the same at session start.

Both are reminders, not blocks. The actual gate is the skill's own rule: no edits until
you approve the digest.

## The database step is destructive

`intake` can restore a `.bak` over your local database. That path runs
`ALTER DATABASE ... SET SINGLE_USER WITH ROLLBACK IMMEDIATE` followed by
`RESTORE ... WITH REPLACE` — active connections are dropped, in-flight transactions
discarded, and the existing database overwritten. Unscripted local schema changes and test
data do not survive it.

The skill always asks first, naming the database and the file. Nothing restores on an
implicit yes.

## Suggested companions

`intake` references `scout` / `implementer` / `reviewer` subagents for delegation after
approval. They are not required — the skill works without them, you just delegate manually.

## License

MIT
