---
description: Send the Claude Code history already on this machine to past.dev
argument-hint: [--all]
---

Send every unsent session already on this machine to past.dev. One session is `/past:ingest <id>`.
The arguments given, if any: `$ARGUMENTS`

1. Run `node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" backfill --current ${CLAUDE_SESSION_ID}` first.
   It sends nothing. It lists each session (date, id, prose size, credits, sittings, first
   prompt) and the total. Each sitting is one data point. The session you are in is always left out: it is sent when it goes idle or ends.
2. Show the user that list and **ask them to confirm**. Sending history to a server is not
   something to do on their behalf. If they want only some of it, `/past:ingest <id>` sends one.
3. Only if they agree, run the same command with `--confirm` added.

`--all` covers every project on the machine instead of this one. Include it in step 1 as well,
so the estimate matches what step 3 will send.
