---
description: Show what past.dev is connected to and what it has sent
---

Run `node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" status` and show the output to the user as it is.

If the key or the identity is missing, say which one and point at `/past:connect`. Do not print
the key itself beyond the prefix the command already shows.

If the user asks to check credits now, add `--refresh` to the status command.
This option reads the balance from the API and clears a credit pause only when the balance is sufficient.
