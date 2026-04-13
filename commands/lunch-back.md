---
description: Resume a saved lunch-break session — list saved sessions or load one
argument-hint: (optional: session number, slug, or partial title)
---

The user has returned from a break and wants to reload context. Saved sessions live in `~/.claude/lunch-break/` as markdown files with frontmatter.

## If $ARGUMENTS is empty — list mode

1. `ls -1t ~/.claude/lunch-break/*.md 2>/dev/null` to list files newest-first.
2. If none exist, say: "No saved sessions yet. Type `/lunch-break` before a break to create one." and stop.
3. Otherwise, for each file (up to the 15 most recent), read its frontmatter and print a numbered list:
   ```
   1. <title>  —  <relative age, e.g. "2h ago">  [<slug>]
   2. ...
   ```
4. Prompt the user: "Reply with a number or slug to load."

## If $ARGUMENTS is provided — load mode

1. Resolve the argument:
   - If it's a number N, pick the Nth most-recent file.
   - Otherwise treat it as a slug or partial title and match against filenames/frontmatter. If multiple match, show the candidates and stop.
2. Read the file in full.
3. Reload context for the user by telling them, in this order:
   - **Title** and when it was saved (relative age)
   - **Goal** (one line)
   - **Done** (bulleted, short)
   - **Ongoing** (bulleted, short — emphasize current state)
   - **Todo** (bulleted, short)
   - **Notes** (only if present and useful)
4. End with a clear prompt:
   > **Next up:** <the Next field, verbatim>
   >
   > Ready to continue? (Confirm and I'll pick up from there.)

## Rules

- Do **not** start acting on the saved plan until the user confirms.
- If the saved `cwd` differs from the current cwd, mention it — the user may have opened cc in the wrong directory.
- If the saved `branch` differs from the current branch, mention it.
- Keep the summary compact. The user just wants to remember, not re-read everything.

$ARGUMENTS
