---
description: Connect this machine to a past.dev project
argument-hint: <project-api-key> <identity> [api-url] [--audience <slug>]
---

Connect Claude Code to past.dev.

The user may have given the key, the identity, an API URL and an audience as arguments:
`$ARGUMENTS`

If a project API key (it starts with `past_sk_`) and an identity are both present, run the command
below, adding the API URL as a third argument when one was given, and `--audience <slug>` when
one was:

```
node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" connect <project-api-key> <identity> [api-url] [--audience <slug>]
```

If only an audience was given (`/past:connect --audience <slug>`), run
`node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" connect --audience <slug>`: it changes who sees what
is sent from now on and keeps the key. `--audience project` goes back to the whole project.

Never repeat the key back, and never write it into a file in the repository.

If the key or the identity is missing (and this is not an audience-only change), do not ask for
the key in the conversation. Tell the user the two ways to
connect instead:

- **This command with its arguments**, wherever Claude Code runs:
  `/past:connect <project-api-key> <identity> [api-url]`. The key is redacted before a session is
  sent to past.dev, but it stays in this machine's local transcript.
- **In the terminal CLI, `/plugin configure past@past`**, a form that masks the key and keeps it
  in the system's credential store. It takes effect at the next prompt or session.

The **audience** decides who in the project sees the sessions sent from this machine. Without
one, everyone in the project does. It is an audience slug from the console (**Audiences**); the
command checks it exists. When the user connects without one, say so in one line and how to
narrow it later: `/past:connect --audience <slug>`.

The **project API key** starts with `past_sk_`. It is created in the past.dev console under
**Build › API keys** and shown once. It is not the project's id or handle, and not the
organization's management key (`past_mk_…`). The identity is
the email or id every memory is attributed to; recall stays off without one. The API URL is only
for a self-hosted or staging deployment.

After it succeeds, tell them two things: new sessions are read from now on, and `/past:backfill`
sends the history already on this machine.
