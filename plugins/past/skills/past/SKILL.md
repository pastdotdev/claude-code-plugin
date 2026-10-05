---
name: past.dev
description: Search this project's own history in past.dev — earlier decisions, why something was built a certain way, what was tried before. Use when the user asks what was decided, whether something was attempted already, or why the code looks the way it does, and the answer is not in the working tree.
---

# Searching past.dev

past.dev holds this team's earlier Claude Code sessions as memory. The repository says what the code
is. past.dev says why it became that, and what was rejected on the way.

## When to search

Search when the question is about history rather than about the current code:

- "What did we decide about X?"
- "Have we tried this before?"
- "Why is this built this way?"
- "What was the reasoning behind Y?"

Do not search for things the working tree answers. Read the files for those.

## How to search

```
node "${CLAUDE_PLUGIN_ROOT}/bin/past-hook.mjs" search <the user's question, in their own words>
```

Pass the question as it was asked. The recall runs on meaning, so a rewritten or shortened
question returns worse evidence than the original.

## How to use what comes back

Each result carries a date and the text of one memory. Cite the date when you use one, so the
user can judge how current it is. A memory is evidence of what was said at a time, not a
statement of what is true now — when it disagrees with the code, the code wins and it is worth
telling the user that the two disagree.

If nothing comes back, say so plainly. Do not fill the gap with a guess.
