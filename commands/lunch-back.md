---
description: Resume a saved lunch-break session — list saved sessions or load one
argument-hint: (optional: session number, slug, or partial title)
---

The user has returned from a break and wants to reload context. Saved sessions live in `~/.lunch-break/` as markdown files with frontmatter.

## If $ARGUMENTS is empty — list mode

1. `ls -1t ~/.lunch-break/*.md 2>/dev/null` to list files newest-first.
2. Read frontmatter from each. **Filter out any file whose frontmatter contains an `attached_at` field** — those have already been resumed by another cc session and shouldn't show up again.
3. If no unattached sessions remain, say: "No saved sessions yet. Type `/lunch-break` before a break to create one." and stop.
4. From the unattached files, take the 3 most recent and read title / slug / saved_at.
5. Use the **AskUserQuestion** tool so the user can pick with arrow keys. Build one question:
   - `question`: "Which saved session do you want to resume?"
   - `header`: "Session"
   - `multiSelect`: false
   - `options`: one per recent file, up to 3. Each option:
     - `label`: the title (truncate to ~60 chars if needed)
     - `description`: `<relative age> · [<slug>]`
   - cc automatically appends an "Other" choice — the user can type a number or slug there to pick an older save. Do **not** add your own "Other" option.
6. When the user picks an option, match it back to the file by slug and proceed to load mode. If they chose "Other" and typed input, treat that input as $ARGUMENTS and re-enter load mode.

## If $ARGUMENTS is provided — load mode

1. Resolve the argument:
   - If it's a number N, pick the Nth most-recent file.
   - Otherwise treat it as a slug or partial title and match against filenames/frontmatter. If multiple match, show the candidates and stop.
2. Read the file in full.
3. **Mark it attached** by adding an `attached_at: <ISO-8601 local timestamp>` line to the frontmatter (edit the file in place, right after `saved_at:`). This hides it from future `/lunch-back` lists so no other cc session will try to resume the same handoff. The file is kept for history — explicit loads by slug still work.
4. Reload context for the user by telling them, in this order:
   - **Title** and when it was saved (relative age)
   - **Goal** (one line)
   - **Done** (bulleted, short)
   - **Ongoing** (bulleted, short — emphasize current state)
   - **Todo** (bulleted, short)
   - **Notes** (only if present and useful)
5. End with a clear prompt:
   > **Next up:** <the Next field, verbatim>
   >
   > Ready to continue? (Confirm and I'll pick up from there.)

## Rules

- Do **not** start acting on the saved plan until the user confirms.
- If the saved `cwd` differs from the current cwd, mention it — the user may have opened cc in the wrong directory.
- If the saved `branch` differs from the current branch, mention it.
- Keep the summary compact. The user just wants to remember, not re-read everything.

$ARGUMENTS
