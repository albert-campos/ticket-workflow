---
name: intake
description: Jira ticket intake gate. Fetches ticket context, checks local database freshness, and produces an approval digest before any code is written. Use when a ticket ID (PREFIX-####, prefix_####, prefix####) appears in a prompt or the current git branch.
---

# Ticket intake gate

Nothing in the working tree changes until the user approves the digest at the end of this
skill. That is the whole point of the gate.

## Configuration

Read these from the environment; fall back to the default when unset.

| Variable | Default | Purpose |
|---|---|---|
| `TICKET_PREFIX` | `ISI` | Jira project key. Matched case-insensitively, with `-`, `_`, or nothing between prefix and number. |
| `DB_BACKUP_DIR` | `/c/Users/Public/Downloads` | Where dropped `.bak` files land. |
| `SQL_SERVER` | `localhost` | `sqlcmd -S` target. |
| `TARGET_DB` | *(unset)* | Database the `.bak` restores into. If unset, ask which database before restoring. |

Set them per machine in `~/.claude/settings.json` under `env`, or per repo in
`.claude/settings.json`. The database step never runs without the user's explicit yes
either way.

## 1. Fetch the ticket

Extract the ticket ID and normalize it to `<PREFIX>-####` for the Jira lookup - branch
forms (`abc_4809`, `abc2825_...`) and prose forms (`abc5879`) all refer to the same key.
`<PREFIX>` is `$TICKET_PREFIX` (see Configuration above).

Use `mcp__plugin_atlassian_atlassian__getJiraIssue`. Pull description, acceptance criteria,
status, assignee, and the **full comment thread** - the comments routinely carry scope
changes that never made it into the description.

If the ticket is not found, say so and stop. Do not infer requirements from the branch name.

## 2. Check local database state

Read-only first. Never restore without confirmation.

```bash
ls -la "${DB_BACKUP_DIR:-/c/Users/Public/Downloads}"/*.bak 2>/dev/null
```

- **A `.bak` is present** - report its filename, size, and modified date. Then check what it
  would overwrite before proposing anything:

  ```bash
  sqlcmd -S "${SQL_SERVER:-localhost}" -Q "SET NOCOUNT ON; SELECT name, state_desc, (SELECT MAX(restore_date) FROM msdb.dbo.restorehistory r WHERE r.destination_database_name = d.name) AS last_restore FROM sys.databases d WHERE database_id > 4;"
  ```

  Report the target database and its last restore date. **Ask for confirmation, naming the
  database and the file.** Only on an explicit yes, run:

  ```sql
  ALTER DATABASE [<DB>] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
  RESTORE DATABASE [<DB>] FROM DISK = '<DB_BACKUP_DIR>\<FileName>' WITH REPLACE;
  ALTER DATABASE [<DB>] SET MULTI_USER;
  ```

  Then rename the file to `<FileName>.processed` so it is not restored again.

  `ROLLBACK IMMEDIATE` disconnects active sessions and discards in-flight transactions;
  `WITH REPLACE` overwrites the existing database. Unscripted local schema changes and test
  data are lost. This is why it asks.

- **No `.bak` present** - query `msdb.dbo.restorehistory` for the last restore. Older than
  7 days, flag it as **Stale**; otherwise **Fresh**.

- **`sqlcmd` unavailable or SQL Server unreachable** - report `Unknown`. Do not treat an
  unreachable server as a fresh database.

## 3. Output the digest, then stop

Use this markdown layout exactly. It is read on a terminal, so it must survive wrapping.

```markdown
## <KEY> — <ticket summary>

**Tier <1|2|3>** · DB: <Fresh (<date>) | Stale (<n>d) | Restored <file> | Unknown> · Jira: <status>

### Goal
<2-3 sentences max. What "done" looks like. If scope changed in the comments, lead with the
change and date, not the original ask.>

### Acceptance
<Bulleted criteria, or "None stated — <why>". Never invent criteria.>

### From the comments
<Only what changes the work. One bullet per point, each a single line. Resolved history that
no longer affects the outcome gets one summary bullet, not a recap.>

### Unknowns — answer before work starts
1. <One question per line. State the decision needed, not the background.>
2. <If a question is blocked on access or permission, say what you need from the user.>

### Not this ticket
<Unrelated working-tree state, untracked files, adjacent bugs. One line each. Omit the
section when there is nothing.>

**Proceed with step 1?**
```

Formatting rules that matter more than the template:

- **One line per idea.** Never wrap a paragraph inside a label column.
- **Unknowns are the payload.** Each numbered, each independently answerable. If an unknown
  is really a blocked action, say who has to unblock it.
- Put dates inline with the claim they support (`scope reversed 2026-09-21 — Ryan Horton`),
  not in a trailing clause.
- Keep `Goal` under three sentences. Detail belongs in the comments section.
- If ticket text arrived garbled or truncated, say so on its own line and offer to re-fetch.
  Do not paper over it mid-sentence.
- Omit a section entirely rather than writing "none" three times.

Then stop and wait. Do not open source files, propose a patch, or start work.

Ask the unknowns now — answering them before coding is the cheapest they will ever be.

## 4. After approval

- Delegate exploration to `scout`, implementation to `implementer`, post-change audit to
  `reviewer`.
- Run `offload` before the context fills or before switching tickets.
