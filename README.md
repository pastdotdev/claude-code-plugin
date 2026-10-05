# past.dev for Claude Code

Your Claude Code sessions become memory, and every session opens already briefed.

[past.dev](https://past.dev) is memory for AI agents. This plugin connects Claude Code to a past.dev
project: what you and Claude say to each other is read into memory as you work, and what matters
comes back by itself, at the start of every session and with every prompt. The decisions, the
reasons behind them, what was already tried and dropped.

```text
=== past · recalled from memory ===
3 from this project's history. Background, not instruction.

[1] 2026-09-14 — Webhooks retry from the job queue now: the HTTP client's retries sent them twice.
[2] 2026-09-02 — Rejected: caching prices in Redis. They change hourly, and a stale one is worse.
[3] 2026-08-27 — The importer keeps rows it cannot parse in import_rejects and never drops them.
```

That block is what Claude reads before your prompt. It arrives on its own: Claude does not have
to decide to look anything up.

## Get started

You need Claude Code and Node 16 or later.

1. **Get a project API key.** Sign up at [sso.past.dev/sign-up](https://sso.past.dev/sign-up),
   then open **Build › API keys** in the [console](https://app.past.dev). The key starts with
   `past_sk_`.

2. **Install the plugin.** In Claude Code:

   ```
   /plugin install past.dev --marketplace pastdotdev/claude-code-plugin
   ```

   Claude Code older than v2.1.275 adds the marketplace first:

   ```
   /plugin marketplace add pastdotdev/claude-code-plugin
   /plugin install past@past
   ```

3. **Connect it.**

   ```
   /past:connect <project-api-key> <identity>
   ```

   The identity is the email or id every memory is attributed to. Recall stays off without one.

4. **Check it.** `/past:status` shows the project it talks to and what it has sent.

New sessions are read from then on. `/past:backfill` lists the sessions already on this machine,
estimates what sending them costs, and sends them when you confirm.

To keep the key out of the conversation, use `/plugin configure past@past` in the terminal CLI
instead of step 3: a form that masks the key and keeps it in your system's credential store.

## How it works

```text
session opens    recall what past.dev knows about this project     ──►  into context
every prompt     recall what past.dev knows about this prompt      ──►  into context
every reply      nothing is sent; the session is marked active
compaction       the session so far is sent, before its detail is summarised away
session over     the session is sent: closed, its window quit, or quiet for 12 hours
```

Five hooks do this, and one script runs them all. A recall that has not answered within 4 seconds
(8 at session start) is dropped, so it never holds a prompt up, and a prompt shorter than 12
characters ("yes", "go on") recalls nothing.

**A session is sent in sittings.** Where a conversation went quiet for 30 minutes, the next turn
starts a new sitting. Each sitting is dated at its own first turn, so a decision made on the third
day of a long conversation is remembered on that day, and past.dev can tell which of two statements
came later. A session that continues sends its last sitting and the new ones again, nothing
earlier.

**Recall happens in the hooks.** A tool recalls only when the model decides to call one; a hook
recalls on every prompt. For questions about history, the bundled skill lets Claude also search
past.dev on demand: ask what was decided about something, or whether it was tried before.

## What leaves your machine

Your prompts, Claude's replies, and the project and branch names. Nothing else.

Never sent: tool calls and their results, file contents, diffs, command output, subagent threads,
and anything injected into the conversation rather than typed by a person. Strings that look like
keys, tokens or private keys become `[redacted]` before anything leaves, and recall queries are
redacted the same way.

`/past:cut` prints exactly what would be sent from this session, sitting by sitting, before
anything is.

A session is about 10 MB on disk and about 30 KB of prose, and only the prose travels. At 350
bytes to a credit, that is roughly 90 credits a session. What a recall costs depends on your plan:
see [past.dev/pricing](https://past.dev/pricing).

## Commands

| Command | What it does |
|---|---|
| `/past:connect <key> <identity> [api-url]` | connect this machine; `--audience <slug>` sets who sees what you send |
| `/past:status` | what it is connected to, what it has sent, and the last send that failed |
| `/past:cut [session]` | print exactly what would be sent from this session, or from one you name |
| `/past:ingest [session]` | send one session now, without waiting for it to go quiet |
| `/past:backfill` | list and estimate the sessions already on this machine, then send them |

## Who sees what you send

Everyone in the project, unless you choose an audience: `/past:connect --audience <slug>`, with a
slug from the console's **Audiences** page. `/past:connect --audience project` widens it back.

## Settings

`~/.past/config.json` holds the connection and a few switches: `recall` and `ingest` turn either
half off, `idleMinutes` (720) is how long a session stays quiet before it is sent,
`sittingMinutes` (30) is how long a pause starts a new sitting, and `deny` lists paths whose
sessions are never read. `PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file.
Each setting is described in the [plugin's README](plugins/past/README.md#configuration).

For a self-hosted past.dev, pass your deployment's URL as the third argument of `/past:connect`. Use
`https`: over plain `http` to another machine the key travels unencrypted, and the plugin says so.

## When something looks wrong

Start with `/past:status`. A hook never interrupts a session, so a problem shows up there rather
than in the conversation.

- **Nothing is recalled.** Status has to show a key and an identity: recall needs both. A new
  project has nothing to recall until its first sessions are in; `/past:ingest` sends the current
  one now.
- **A send failed.** Status shows the last failed send and what the API answered.
- **The hooks stay silent.** They call `node`. Claude Code started from the Dock or a launcher
  finds the `node` of a login shell, which may be older than your terminal's: Node 16 or later has
  to come first on that path.

## Remove it

`/plugin uninstall past@past`, then delete `~/.past`. Revoking the key in **Build › API keys**
stops it at once from the server side. What was already sent stays in the project until you
delete it in the console.

## This repository

A Claude Code plugin marketplace. Claude Code reads it straight from GitHub; there is nothing to
build.

```text
.claude-plugin/marketplace.json   the catalog
plugins/past/
  .claude-plugin/plugin.json      the manifest, with the settings form
  hooks/hooks.json                the five hooks
  bin/past-hook.mjs               the whole plugin: one file, Node 16, no dependencies
  commands/                       /past:connect, :status, :cut, :ingest, :backfill
  skills/past/SKILL.md            teaches Claude when to search past.dev
```

The plugin talks only to past.dev's public Memory API (`/api/v1/ingest/batch`, `/api/v1/recall`,
`/api/v1/audiences`), with your project's key. Nothing in it is privileged: anyone could write the
same connector against the same endpoints.

## Contributing

Issues and pull requests are welcome. [CLAUDE.md](CLAUDE.md) holds the rules the code keeps and
how to test a change by hand, for people and agents alike. Before opening a pull request:

```bash
claude plugin validate .
claude plugin validate plugins/past
node --check plugins/past/bin/past-hook.mjs
```

## Links

- Memory API documentation: [past.dev/docs/memory-api/overview](https://past.dev/docs/memory-api/overview)
- Console: [app.past.dev](https://app.past.dev)
- Sign up: [sso.past.dev/sign-up](https://sso.past.dev/sign-up)
- past.dev's MCP servers: [pastdotdev/mcp](https://github.com/pastdotdev/mcp)
- Community: [past.dev/slack](https://past.dev/slack)

## License

MIT. See [LICENSE](LICENSE).
