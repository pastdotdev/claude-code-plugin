# past.dev for Claude Code

Your Claude Code sessions become memory, and every session opens already briefed.

## Install

```
/plugin marketplace add pastdotdev/claude-code-plugin
/plugin install pastdotdev@pastdotdev
```

The first line adds this repository as a marketplace. The second line installs the plugin from it.

In the Claude desktop app, or on claude.ai: open **Customize › Plugins**, select **Add › Add
marketplace** and enter `pastdotdev/claude-code-plugin`, then open **past.dev** and select **Add**.
The plugin is saved to your account and loads in Claude Code, in the app's Code tab and in the
terminal, from the next session on; `/reload-plugins` loads it into a session already open.

Then connect it, wherever Claude Code runs:

```
/pastdotdev:connect <project-api-key> <identity> [api-url]
```

The project API key starts with `past_sk_` and comes from **Build › API keys** in the
[console](https://app.past.dev) (not the project's id, and not the organization's `past_mk_`
management key); the identity is who every memory is attributed to, and the API URL is for
self-host only. In the terminal CLI, `/plugin configure pastdotdev@pastdotdev` does the same through a form
that masks the key and keeps it in your system's credential store; it takes effect at your next
prompt or session.

## What it does

| Hook | When | What |
|---|---|---|
| `SessionStart` | a session opens | recalls this project's history and puts it in context |
| `UserPromptSubmit` | every prompt | recalls the ten memories closest to the prompt and puts them in context |
| `Stop` | after each reply | starts or resets the idle timer; sends nothing itself |
| `PreCompact` | the context is about to be compacted | sends the sittings that closed since the last send; the one still open at most once an hour |
| `SessionEnd` | a session closes | sends the session's prose to past.dev |

A session also counts as over after 12 hours without a new turn (`idleMinutes` in
`~/.past/config.json`), and is sent then. That is what gets a desktop-app conversation, which can
stay open for days, into past.dev; if it continues, it is sent again at its next pause. Closing the
window instead of the conversation (quitting VS Code or the app) skips `SessionEnd`; the plugin
notices the conversation's process is gone and sends it within a minute.

**A session is sent in sittings.** A new sitting starts where the session went quiet for 30
minutes (`sittingMinutes`): nobody wrote and the agent ran nothing. One also starts once the
sitting so far holds 50 KB of prose (`sittingKilobytes`), after the agent's reply, so a session that
never pauses is cut all the same. Each sitting is dated at its own
first turn, and what past.dev remembers takes the date of the sitting it came from, so something said
on the third day of a conversation that ran for three days is dated that day. A prompt and its reply stay
together while the agent works on it, however long. When a session continues, only its last sitting
and the new ones are sent again.

`SessionStart` also sends any earlier session that never reached past.dev and has gone idle, which is
what covers a crash or a machine that slept. `/pastdotdev:status` counts sessions as *sent* and *open*;
an open session leaves the list by itself once it is in past.dev unchanged and idle, or holds nothing
to send.

**Compaction is the moment this matters most.** When the window fills, Claude Code summarises
everything before the boundary and drops the detail. `PreCompact` sends the sittings that closed since
the last send, and the one still open at most once an hour, since the transcript on disk keeps every
turn; `SessionStart` then fires again with source `compact`, and its recall is aimed at
what this session was doing rather than at the project at large — which is exactly what the model
has just lost.

## What leaves this machine

The prompts, the replies, and the project and branch names. That is all.

Not sent: tool results, file contents, diffs, command output, subagent threads, and anything
the agent injected rather than a person typing it. Strings that look like keys, tokens or
private keys are replaced with `[redacted]` before anything is sent.

Run `/pastdotdev:cut` to read exactly what would be sent, before you send it.

A session is about 10 MB on disk and about 30 KB of prose. past.dev sends the prose. At 350 bytes
to the credit that is roughly 90 credits for a session.

## Commands

| Command | Does |
|---|---|
| `/pastdotdev:connect` | store the key, identity and audience in `~/.past/config.json`, mode `600` |
| `/pastdotdev:status` | what it is connected to, what it has sent, and the last send that failed |
| `/pastdotdev:cut [session]` | print exactly what would be sent from this session, or from one you name |
| `/pastdotdev:ingest [session]` | send one session now — this one, or one you name |
| `/pastdotdev:backfill` | list and estimate, then send, every unsent session already on this machine |

Without an audience everyone in the project sees what you send. `/pastdotdev:connect --audience <slug>`
narrows it to an audience from the console; `--audience project` widens it back.

**And a recall tool.** The plugin also serves `past_recall`, on a small MCP server of its own
(`.mcp.json`, run by the same script). The block before a prompt holds what was recalled for it;
when the request depends on something that block does not hold, Claude calls `past_recall` with a
question of its own and reads up to a hundred dated memories, within 2,000 tokens. Each call is one
recall. Claude Code asks you before the first call: allow it, for the session or for good.
`claude mcp list` shows the server as `plugin:pastdotdev:past`.

The bundled skill teaches Claude when to look history up: ask what was decided, or whether
something was tried before, and it looks.

## Configuration

`~/.past/config.json`:

```json
{
  "apiKey": "past_sk_…",
  "identity": "you@example.com",
  "apiUrl": "https://api.past.dev",
  "audience": "developers",
  "recall": true,
  "ingest": true,
  "idleMinutes": 720,
  "sittingMinutes": 30,
  "sittingKilobytes": 50,
  "deny": ["/Users/you/clients/acme"]
}
```

- `audience` is an audience slug from the console; leave it out and the whole project sees what
  you send.
- `recall: false` stops the reading and keeps the sending. `ingest: false` does the opposite.
- `idleMinutes` is how long a session stays quiet before it is sent (720, twelve hours, by default).
- `sittingMinutes` is how long a session stays quiet before what follows starts a new sitting (30
  by default, 5 at least). Shorter dates what past.dev remembers more closely, in more and smaller
  data points. A session keeps the length it was first sent with.
- `sittingKilobytes` is how much prose a sitting holds before the next turn starts a new one (50
  by default, 10 at least), the cut waiting for the agent's reply. A session keeps the size it was
  first sent with.
- `deny` holds path fragments. A session whose working directory matches one is ignored
  completely — nothing is read and nothing is sent.
- `PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file.
- Self-host: point `apiUrl` at your own deployment. Nothing else changes. Use `https`: over plain
  `http` to another machine the key travels unencrypted, and `connect` and `status` warn about it.

## Requirements

Node 16 or later. The hooks call `node` directly, so Windows works.

## Where it keeps things

Everything lives in `~/.past/`, a folder only you can open: `config.json` (the connection),
`state.json` (what was sent, as content hashes) and `activity/` (one small file per open session,
for the idle timer).

## Moving from past@past

Before 0.3.0 the plugin was installed as `past@past`. To move to the new name, run these in Claude Code:

```
/plugin uninstall past@past
/plugin marketplace remove past
/plugin marketplace add pastdotdev/claude-code-plugin
/plugin install pastdotdev@pastdotdev
```

Your connection stays in `~/.past`, so you don't need to connect again, and sessions already sent aren't sent twice.

## Removing it

`/plugin uninstall pastdotdev@pastdotdev`, then delete `~/.past`. Revoking the key in **Build › API keys**
stops it immediately from the server side. What was already sent stays until you delete it in the
console.

## License

MIT — see [LICENSE](LICENSE).

## Credits

After `402 out-of-credits`, the plugin pauses the refused operation and keeps unsent data locally.
A write refusal does not pause recall. A recall refusal pauses recall separately.
On later calls, the plugin checks the balance at most once per five minutes for each paused operation.
The plugin resumes when the balance covers the refused call. The server checks the balance again for each request.
Run `/pastdotdev:status --refresh` after you add credits to check the balance now.
A new API key or API URL starts with no pause from the previous connection.
