# past.dev for Claude Code — working on the plugin

The past.dev plugin for Claude Code, and the marketplace that carries it. It runs on the developer's
machine and talks to past.dev's public Memory API. Nothing here is privileged: anyone can
write the same connector against the same endpoints with the same key. What a user reads is in
`README.md`, and in full in `plugins/past/README.md`; this file is the rules the code keeps, for
whoever changes it.

## How it fits together

```
Claude Code
  hooks/hooks.json
  commands/*.md, skills/past
              |
            bin/past-hook.mjs <mode>
              |
  session start   brief        recall into context; sends idle sessions left behind
  every prompt    prompt       recall into context
  every turn      stop         stamps the activity file, keeps one waiter on the session
  compaction      precompact   sends the session so far
  session end     end          hands the send to a detached process (flush)
  detached        idle         the waiter: sends once Claude Code's process is gone, or after idleMinutes
              |
  ~/.past/config.json          the connection
  ~/.past/state.json           what was sent and what is open
  ~/.past/activity/            one file per open session
              |
  POST /api/v1/recall · POST /api/v1/ingest/batch · GET /api/v1/audiences/{slug}
```

```
.claude-plugin/marketplace.json   the catalog — this repository is the marketplace
plugins/past/
  .claude-plugin/plugin.json      the manifest, including the settings form (userConfig)
  hooks/hooks.json                SessionStart, UserPromptSubmit, Stop, PreCompact, SessionEnd
  bin/past-hook.mjs               the whole connector, one file, no dependencies
  commands/                       /past:connect, :status, :cut, :ingest, :backfill
  skills/past/SKILL.md            teaches Claude when to search history
  LICENSE                         the same license: an install copies this folder and nothing else
.github/workflows/check.yml       every pull request: the script parses and runs, the manifests validate
```

## The rules this code keeps

1. **A hook never breaks the session.** Every mode exits 0, prints valid JSON or nothing, and
   swallows its own failures. `/past:status` is where a person learns something is wrong, so a
   failed send is kept in `state.lastError` for it.
2. **Only prose travels.** A session is ~10 MB on disk and ~30 KB of prose. Tool results, file
   contents, diffs, subagent threads and harness-injected turns are cut. Sending the transcript
   would cost about three hundred times more and recall worse.
3. **The hash gates the send.** `state.json` holds, per session, the hash of the whole session
   rendered in one piece (`hash`) and one hash per sitting (`sittings`). A session whose whole
   hash has not changed is not sent; one that changed sends only the sittings whose hash differs.
   The server charges nothing for an unchanged re-send, but the call still spends a request from
   the key's per-minute budget. Do not remove it.
4. **Send when a session ends, goes idle, closes or compacts; never from `Stop` itself.** `Stop`
   fires every turn, and a session that grew sends its last sitting again as a new revision.
   - `Stop` only stamps `~/.past/activity/<session>.json` and keeps one detached waiter (`idle`
     mode) per session. The waiter sends once the last turn is `idleMinutes` old (720 by
     default), measured on the wall clock: it sleeps in one-minute slices, because a timer does
     not count the time the machine is asleep, and one long timer fired about eight hours late
     after a night with the lid closed.
   - `Stop` also records the session's owner: the agent's process above the hook, shells skipped.
     Claude Code has one per conversation, in the CLI, the desktop app and the VS Code extension.
     Closing a window or quitting the app kills it without a `SessionEnd`; the waiter sees it gone
     within the minute and sends the session as ended. There is no `ps` on Windows, so no owner
     there, and the idle timer alone applies.
   - `ensureWaiters` (`Stop`, `SessionStart`, `/past:status`) replaces a waiter that died or that
     an older `WAITER_VERSION` started — bump it whenever the waiter changes — and a replacement
     past its deadline sends at once. A waiter that a newer copy started is left alone.
   - In Claude Code, `SessionEnd` clears the stamp and detaches the send (`flush` mode), because
     Claude Code gives exit hooks about 1.5 s and plugin timeouts do not extend it.
   - `PreCompact` flushes before the context is summarised; the session stays *pending*, as it
     does after an idle send. After a compaction `SessionStart` fires again with source
     `compact`, and there the recall query is this session's own last prompts.
   - `tidyPending` (status and every session start) drops a pending session whose transcript is
     gone or empty, or that is in past.dev unchanged and idle; `Stop` re-adds one that resumes.
     `SessionStart` sends idle sessions that never ended cleanly, never one still being written
     in another window.
5. **Redact the query, not only the content.** A recall query crosses the network exactly as
   ingested content does. Both go through `redact()`. A prompt that pastes a key must not put that
   key into a query string.
6. **Backfill estimates before it sends, and never sends the session it runs in.** `backfill`
   lists; `backfill --confirm` sends them all. The current session is left out: the hooks send
   it, and a second send would pay for it twice. The command file names it (`--current
   ${CLAUDE_SESSION_ID}`).
   One session is `ingest`, which sends at once — running it is the consent — and may send the
   current one, which then stays pending. `cut` defaults to the current session, then to the most
   recently written transcript, never the largest.
7. **Two ways in, one store.** `/past:connect` works wherever Claude Code runs and is the one the
   docs lead with. The settings form (`userConfig`, opened by `/plugin configure past@past` in the
   terminal CLI only; the key masked and kept in the system credential store) is the alternative.
   Both end in `~/.past/config.json`, which every mode reads. Form values reach
   hooks only, as `CLAUDE_PLUGIN_OPTION_*`, so they land at the next hook, not when the form
   closes; commands run through Bash never see them. A form value is copied when it changed since
   the last copy (`state.json` keeps its hash) or the file lacks it, so whichever changed last wins
   and a deleted file is restored. Clearing a form field clears nothing. Commands and the skill
   call the script through `${CLAUDE_PLUGIN_ROOT}`, since only a bare file in `bin/` is on PATH and
   this one ends `.mjs`.
8. **`state.json` changes only through `updateState`.** Hooks and waiters are separate processes
   running at once. `updateState` takes a lock (`~/.past/state.lock`, a directory, taken over
   after 5 s, skipped after 3 s so a hook never hangs), re-reads the file, applies the change and
   renames the new file into place. Two waiters that sent together once each saved a copy read
   before their send, and the second erased the first one's "sent". Never `saveState` a copy. A
   file that does not parse is read again a few times and, if it still does not, left as it is:
   written over, it would lose every agent's sessions, and the next backfill would send them all.
9. **No dependencies.** Node 16 or later, standard library only, so install is `git clone` and
   Windows works. The hooks call `node` directly rather than relying on a shebang. `fetch` is used
   where Node has it and the `http` modules where it does not: an agent started from the Dock
   finds the Node of a login shell, which is often an old one, and a call that throws there fails
   without a sound. On Node 16 an `http` request stops on its signal only while its body is being
   written, so the fallback destroys the request itself when its time is up; a stalled call would
   otherwise hold a prompt, or a waiter's send, for ever. Nothing in the script is newer than Node
   14 can parse (no `||=`), so on an older Node the hooks stay silent rather than fail.
10. **What Claude Code sends does not change by a byte.** The heading and the speaker names that
    `renderTranscript` prints are part of the content hash. A changed byte makes `backfill` send
    every session already in past.dev again, as a paid revision. The same holds for `SOURCE`, `AGENT`
    and `SPEAKER` at the top of the script. `sittingId` and the cut (`sittingsOf`,
    `silenceMeter`) are fixed for a related reason: changed, they move the boundaries of every
    session that grows afterwards, its sittings are sent again under their new ids, and the old ones
    stay in past.dev beside them. A person's `sittingMinutes` cannot do that (rule 12).
11. **`~/.past` is the person's alone, and the key never travels in the clear unnoticed.** The
    folder is made `700` and every file in it is created `600` (`privateDir`, `writePrivate`):
    never written open and narrowed afterwards, since a file that holds the key would be readable
    in between. An `apiUrl` over plain http, on a host other than this machine, sends the key
    unencrypted: `connect` warns and `status` says so on its first line. It is a warning, since a
    self-hosted deployment on a private network may have no certificate.
12. **A session is sent in sittings.** past.dev dates every memory at the time of the data point it
    was drawn from, and decides which of two statements is the later from those times, so the
    plugin cuts a session where it went quiet. A sitting starts at the first turn after the
    transcript was silent for `sittingMinutes` (30 by default, 5 at least), whoever speaks: a
    resumed session ("Continue from where you left off") or a permission approved the next morning
    starts with the agent. Silence is measured over the conversation's own entries, tool calls and
    their results included (`silenceMeter`), and over nothing else: other lines, such as a pull
    request's status, are written while nobody is there. So a prompt stays with its reply while the
    agent works; one tool call that writes nothing for longer than a sitting is the case that parts
    them. Each sitting is one data point timed at its first turn. A session only grows at its end,
    so a sitting never changes once the next has begun.
    - `state.json` pins `sittingMinutes` per session at its first send. A changed value applies to
      the sessions sent after it: moving the boundaries of one already in past.dev would send its
      sittings again under shifted ids and leave the old ones beside them.
    - The first sitting keeps the id a whole session had (`claude-code:<sessionId>`); the others
      are `claude-code:<sessionId>:<n>`. A session sent whole before sittings existed is left
      alone while it is unchanged (its whole hash still matches), and when it grows its first
      sitting replaces it under the same id, with no copy left behind.
    - Every changed sitting goes in one `/api/v1/ingest/batch` call, so a session spends one
      request from the per-minute budget however many sittings it has.
    - `cut` prints every sitting under its own heading and prices what a send would carry now;
      `backfill` counts sittings, since each is one ingestion toward the project's monthly cap.

13. **The product is written past.dev.** Every sentence a person reads says past.dev: the READMEs,
    the manifests' display names, the commands' and the skill's text, what a mode prints.
    Identifiers keep `past`: the plugin and marketplace names (`past@past`), the commands
    (`/past:connect`), `~/.past`, the key prefix `past_sk_` and the data point ids. So does the
    recall block's header, `=== past · recalled from memory ===`: the script drops a block by that
    exact text from the sessions it sends, so a changed header would change how old sessions render
    and send them all again (rule 10).
14. **A hook answers Claude Code's own events alone.** Another agent can load this plugin's hooks
    and run them on events of its own, about a conversation that is not Claude Code's. Claude Code
    names the event in every hook's input (`hook_event_name`), so each hook mode checks that it is
    the one it was written for (`SessionStart`, `UserPromptSubmit`, `Stop`, `PreCompact`,
    `SessionEnd`) and otherwise stays silent: no recall is spent on that agent's prompts, and none
    of its conversations is booked as a session. What Claude Code's own sessions send is unchanged.

## Beside other past.dev plugins

`~/.past` can be shared with another past.dev plugin on the same machine. `config.json` is then one
connection for both. In `state.json` this plugin's sessions are at the top, and another plugin's
sit under `hosts`: they are never read here, and every write puts them back as they were, which is
one more reason `state.json` changes only through `updateState` (rule 8). In `activity/` this
plugin's files are at the top; a folder in it belongs to another plugin, and the waiters only look
at `.json` files.

## Why the API and not MCP

Hooks are shell commands and cannot speak MCP. Recall through a hook is injected on every prompt;
recall through an MCP tool happens only if the model chooses to call it. The guarantee is the
product, so the hooks use `/api/v1/recall` and `/api/v1/ingest/batch`.

MCP stays the right surface for Claude web and desktop, and for conversational lookups. Adding a
`.mcp.json` here later is additive and needs the project's MCP server enabled first.

## Contracts this depends on

- `POST /api/v1/ingest/batch` — one item per sitting. `id` gives replacement, `timestamp` dates
  every memory drawn from the item, `label` names the session, `metadata` carries
  `conversationId`, `sessionId`, `project`, `gitBranch`, `sitting`. A batch takes up to 1,000
  items and 16 MB of content.
- `POST /api/v1/recall` — **`identity` is required**. Recall stays off until `/past:connect`
  sets one. The response is one result per document in rank order; the hooks read each result's
  `content` and `occurredAt`.
- `audience` on ingest — a slug; absent means the whole project. `connect --audience` checks it
  with `GET /api/v1/audiences/{slug}` (404 when unknown); a later `audience-unknown` refusal is
  kept in `state.lastError` for `/past:status`, like every failed send.
- `UserPromptSubmit` gives the prompt as `prompt` on stdin.
- Claude Code gives a hook `session_id`, `transcript_path`, `cwd` and `hook_event_name`, and
  takes `hookSpecificOutput.additionalContext` at session start and on a prompt.
- A project API key (`past_sk_…`) names its own project. The plugin never sends a project id.

If any of these contracts changes, this file changes with it.

## Testing

By hand: there is no test suite. `claude plugin validate .` checks the marketplace and
`claude plugin validate plugins/past` the plugin's manifest; CI runs both on every pull request,
with the script parsed on Node 14 and run on Node 16 and 22 (rule 9). The two checks that matter,
from the repository root:

```bash
node plugins/past/bin/past-hook.mjs cut          # what would be sent, from a real transcript
node plugins/past/bin/past-hook.mjs backfill     # the estimate, sending nothing
```

Feed a crafted transcript to `cut` when changing the filter or the
redaction, and confirm that tool results, sidechains, synthetic turns and secret-shaped strings are
all absent.

For the hooks and the waiter, point a throwaway home at a local stand-in for the API, so nothing
touches your real `~/.past` or a real project: `HOME=/tmp/past-test` with a `config.json` whose
`apiUrl` is `http://127.0.0.1:<port>`, a few lines of Node answering `POST /api/v1/ingest/batch` and
logging what arrives, and the mode under test fed its stdin JSON by hand (`echo
'{"hook_event_name":"Stop","session_id":…,"transcript_path":…,"cwd":…}' | node plugins/past/bin/past-hook.mjs stop`). A
small `idleMinutes` in that config makes the idle timer testable in seconds.

**A change to the script is checked against what it sent before.** Run the old script and the new
one over the same transcripts, each in a throwaway home with its own stand-in, and compare what each
mode prints, what each request carries and what `state.json` keeps, byte for byte (rule 10).
