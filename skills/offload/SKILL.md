---
name: offload
description: Offload active session state into PLAN.md and TODO.md before /clear or a context reset. Use when context is filling up, before clearing, or when switching tickets.
---

# Offload session state

Write what the next session needs to resume cold. Do not summarize the conversation - record the work.

1. Scan this session for: files created or modified, architectural decisions and the reasoning
   behind them, open bugs, blocked items, and anything discovered that is not obvious from the
   code itself.
2. Write or update `PLAN.md` in the workspace root:
   - Ticket / goal
   - What is done (with file paths)
   - What is in progress, and its current state
   - Decisions made and why - this is the part that is expensive to rediscover
   - Known problems and dead ends already ruled out
3. Write or update `TODO.md` in the workspace root: the next 3 sequential steps, concrete
   enough to act on without re-reading the plan.
4. Ensure `PLAN.md`, `TODO.md`, and `*.bak.processed` are in the repo's local `.gitignore`.
   Add them if missing.
5. Output exactly: `Memory successfully offloaded. Safe to /clear.`

Record absolute file paths. A future session has none of this conversation.
