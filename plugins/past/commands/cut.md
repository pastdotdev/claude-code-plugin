---
description: Show exactly what past.dev would send from this session, or from one you name
argument-hint: [session-id | transcript-path]
---

Show what would leave the machine for one session. It sends nothing.

If the user named a session (an id, or its first characters) or a transcript path, run:

```
node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" cut <session-or-path> --session ${CLAUDE_SESSION_ID}
```

Otherwise run it for the session you are in:

```
node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" cut --session ${CLAUDE_SESSION_ID}
```

The arguments given, if any: `$ARGUMENTS`

Show the user the result. This is what leaves the machine: the prompts and the replies, with
secrets redacted. Tool results, file contents and diffs are not in it. A session is sent in
sittings, one data point each: a new one starts where the session went quiet for `sittingMinutes`
(30 by default), and each begins with its own heading. The first line names the session and its sittings,
and reports the size on disk, the size sent, and what sending it now costs. The last line says what
happens next and, unless it is already in past.dev, the command that sends this session now
(`/past:ingest`). Relay that line as it is.

Then, unless the last line says the session is already in past.dev, ask the user one yes/no question:
**"Send this session to past.dev now? About N credits."**, with N from the first line. If they say
yes, send exactly the session that was cut, by passing its id from the first line:

```
node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" ingest <session-id> --session ${CLAUDE_SESSION_ID}
```

and relay its output. If they say no, stop there; the session is still sent when it goes idle or
ends.
