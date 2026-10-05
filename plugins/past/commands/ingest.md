---
description: Send one session to past.dev now — the one you are in, or one you name
argument-hint: [session-id | transcript-path]
---

Send one session to past.dev right away, instead of waiting for it to go idle or end. It is usually
the session just read with `/past:cut`. The arguments given, if any: `$ARGUMENTS`

If the user named a session (an id, or its first characters) or a transcript path, run:

```
node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" ingest <session-or-path> --session ${CLAUDE_SESSION_ID}
```

Otherwise send the session you are in:

```
node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" ingest --session ${CLAUDE_SESSION_ID}
```

The user asked for this send by running the command, so do not ask again. Relay the output as it
is: what was sent and its cost, or why nothing was. A session still open is sent again when it
goes idle or ends if it grows, and the output says so. To see the content first, `/past:cut`
shows exactly what would be sent.
