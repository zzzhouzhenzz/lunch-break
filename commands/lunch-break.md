---
description: Save current session context before a break so you can resume after forgetting
argument-hint: (optional note)
---

The user is stepping away from this session and wants to be able to pick it back up later without having forgotten everything. Your job: write a handoff note to disk so future-them (or a resumed session) can load it via `/lunch-back`.

## Steps

1. **Reflect on the session so far.** Read the conversation from the top. Identify:
   - The real goal (not just the last message)
   - What's actually been completed (files changed, decisions made)
   - What's in-flight right now (started but not finished)
   - What's still to do
   - The single most important next action when they return

2. **Generate a short descriptive title** (3–7 words, kebab-case slug friendly). Examples: `lunch-break-plugin-build`, `fix-cookie-jar-svg-background`, `slack-channel-remote-control-design`.

3. **Pick a filename:** `~/.claude/lunch-break/YYYY-MM-DD-HHMM-<slug>.md` (use the current local date/time).

4. **Dedup before writing.** Check `~/.claude/lunch-break/` for any existing files whose filename ends in `-<slug>.md`. If any exist, they're stale saves of the same work — delete them before writing the new file. (Only the newest save of a given slug is useful.) If you need to pick a slightly different slug to distinguish genuinely different work, do that instead — but default to dedup.

5. **Write the file** with this exact frontmatter shape, then a readable body. Keep each section tight — this is a reminder, not a report.

```markdown
---
title: <human-readable title>
slug: <kebab-case slug>
saved_at: <ISO-8601 local timestamp>
cwd: <current working directory>
branch: <git branch if in a repo, else omit>
---

# <title>

## Goal
<one or two sentences — why this session exists>

## Done
- <concrete completed item, with file:line or artifact when useful>
- ...

## Ongoing
- <started but not finished — include current state and where you left off>
- ...

## Todo
- <not yet started>
- ...

## Next
<the single most important thing to do first when resuming — one line>

## Notes
<anything else that would be painful to re-derive: decisions made, dead ends ruled out, surprising gotchas>
```

6. **Create the directory if needed** (`mkdir -p ~/.claude/lunch-break`), write the file, then confirm to the user with:
   - The saved title
   - The full path
   - One sentence of what to type when they return: `/lunch-back`

## Rules

- Do **not** fabricate progress. If something wasn't actually done, put it under Ongoing or Todo.
- Do **not** dump the whole conversation — summarize.
- If the user passed an argument ($ARGUMENTS), treat it as an extra note to include under Notes.
- If the session is trivial (just greetings, nothing to resume), tell the user there's nothing worth saving and skip writing.

$ARGUMENTS
