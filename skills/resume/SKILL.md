---
name: resume
description: Re-hydrate session context from PLAN.md and TODO.md after a /clear or new session. Use when picking up prior work on a ticket.
---

# Resume from offloaded state

1. Read `PLAN.md` and `TODO.md` from the workspace root. If neither exists, say so and stop -
   do not guess at prior work.
2. Report back, briefly:
   - A 2-sentence summary of where the work stands
   - The 3 items from `TODO.md`, verbatim
3. Stop there. Do not begin work until the user confirms which item to take.

If `PLAN.md` references files, verify they still exist before reporting - the tree may have
moved since the offload.
