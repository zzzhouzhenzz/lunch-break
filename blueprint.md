# Blueprint — build `lunch-break` from scratch

You (another Claude Code session) are about to rebuild a small productivity tool called **lunch-break**. It's a set of Claude Code slash commands that save a session handoff to disk before a break and restore context after.

Follow these steps exactly. Don't improvise structure, don't add tooling that isn't listed, don't create a plugin wrapper. The goal is a tiny, dependency-free drop-in for `~/.claude/commands/`.

The user is taking a coffee break. Work autonomously end-to-end. Only pause if a step truly cannot be completed without them.

---

## 0. Product spec (read before building)

**Problem:** After a long coding session — especially across multiple cc sessions — a break erases the user's mental context. Resuming a cc session reloads the model's context window but not the user's memory. They need a written handoff they can re-read in 30 seconds.

**Behavior:**

- `/lunch-break` (aliases: `/lunch`, `/break`, `/lunchbreak`) — cc reflects on the session, generates a short title, and writes a markdown handoff to `~/.lunch-break/YYYY-MM-DD-HHMM-<slug>.md`. Before writing, it deletes any existing file with the same slug so repeated saves don't accumulate.
- `/lunch-back` (aliases: `/lunch-break-back`, `/back`) — with no args, shows up to 3 most recent *unattached* sessions via `AskUserQuestion` (arrow-key pickable, cc appends an "Other" choice for older saves). With args (number or slug/partial-title), loads that file directly. On load, stamps `attached_at` into the frontmatter so other cc sessions won't see the same handoff in their list.

**Storage:** `~/.lunch-break/` (global, across projects and worktrees).

**Non-goals:** No hooks, no MCP server, no plugin wrapper, no background process, no external deps.

---

## 1. Repo layout

Create a new directory (e.g. `~/code/lunch-break`) with this exact tree:

```
lunch-break/
├── README.md
├── blueprint.md                # optional — copy this file in
├── install.sh                  # chmod +x
└── commands/
    ├── lunch-break.md          # primary save command
    ├── lunch.md                # alias → lunch-break
    ├── break.md                # alias → lunch-break
    ├── lunchbreak.md           # alias → lunch-break
    ├── lunch-back.md           # primary restore command
    ├── lunch-break-back.md     # alias → lunch-back
    └── back.md                 # alias → lunch-back
```

---

## 2. File contents

Each slash-command file is a markdown file with YAML frontmatter. cc picks up the `description` for the palette. `$ARGUMENTS` inside the body is substituted with whatever the user typed after the command.

### 2.1 `commands/lunch-break.md` (primary)

```markdown
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

3. **Pick a filename:** `~/.lunch-break/YYYY-MM-DD-HHMM-<slug>.md` (use the current local date/time).

4. **Dedup before writing.** Check `~/.lunch-break/` for any existing files whose filename ends in `-<slug>.md`. If any exist, they're stale saves of the same work — delete them before writing the new file. (Only the newest save of a given slug is useful.) If you need to pick a slightly different slug to distinguish genuinely different work, do that instead — but default to dedup.

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

6. **Create the directory if needed** (`mkdir -p ~/.lunch-break`), write the file, then confirm to the user with:
   - The saved title
   - The full path
   - One sentence of what to type when they return: `/lunch-back`

## Rules

- Do **not** fabricate progress. If something wasn't actually done, put it under Ongoing or Todo.
- Do **not** dump the whole conversation — summarize.
- If the user passed an argument ($ARGUMENTS), treat it as an extra note to include under Notes.
- If the session is trivial (just greetings, nothing to resume), tell the user there's nothing worth saving and skip writing.

$ARGUMENTS
```

> Note: in the embedded frontmatter block above, indent it four spaces (as shown) so the outer markdown engine doesn't eat it. When you write the actual file on disk, keep the content byte-for-byte, but make sure the inner code fence delimiters are preserved.

### 2.2 `commands/lunch.md`, `commands/break.md`, `commands/lunchbreak.md` (save aliases)

Each file has identical content:

```markdown
---
description: Alias for /lunch-break — save session context before a break
argument-hint: (optional note)
---

Follow the instructions in the `/lunch-break` command exactly (see `commands/lunch-break.md` in this plugin). Treat any arguments below as the optional note.

$ARGUMENTS
```

### 2.3 `commands/lunch-back.md` (primary)

```markdown
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
```

### 2.4 `commands/lunch-break-back.md` and `commands/back.md` (restore aliases)

Each file has identical content:

```markdown
---
description: Alias for /lunch-back — resume a saved session
argument-hint: (optional: session number, slug, or partial title)
---

Follow the instructions in the `/lunch-back` command exactly (see `commands/lunch-back.md` in this plugin).

$ARGUMENTS
```

### 2.5 `install.sh`

```bash
#!/usr/bin/env bash
# Install lunch-break slash commands into ~/.claude/commands/
# Symlinks so `git pull` picks up updates automatically.
set -euo pipefail

SRC="$(cd "$(dirname "$0")" && pwd)/commands"
DST="${HOME}/.claude/commands"
mkdir -p "$DST"

for f in "$SRC"/*.md; do
  name="$(basename "$f")"
  ln -sfv "$f" "$DST/$name"
done

echo
echo "Installed. In cc, try:  /lunch-break   /lunch-back"
```

Make it executable: `chmod +x install.sh`.

### 2.6 `README.md`

Short user-facing doc: what it does, the command list, the saved file shape, install instructions (`git clone` + `install.sh`), uninstall (`rm ~/.claude/commands/{lunch-break,lunch,break,lunchbreak,lunch-back,lunch-break-back,back}.md`), and a short design-notes section stating the three principles: pure slash commands, global storage, model-generated titles.

---

## 3. Build steps (do these in order)

1. `mkdir -p lunch-break/commands && cd lunch-break`
2. Create each file from §2 with the exact content. Do not add code fences, comments, or preamble beyond what's shown.
3. `chmod +x install.sh`
4. `git init && git add -A && git commit -m "Initial lunch-break"` — small commits are fine if you prefer.
5. (Optional) `gh repo create lunch-break --public --source=. --push --description "..."` to publish.

---

## 4. Verification — run these checks before declaring done

These are the concrete checks. Don't skip them.

### 4.1 Structural checks

- [ ] `ls commands/` shows exactly 7 files, names matching §1.
- [ ] Each `.md` file has valid YAML frontmatter (starts with `---`, has `description:`, ends with `---`).
- [ ] `install.sh` is executable: `test -x install.sh`.
- [ ] No `.claude-plugin/` directory exists (this is intentionally **not** a plugin).

### 4.2 Install check

- [ ] Run `./install.sh`. Expect 7 symlinks created in `~/.claude/commands/`.
- [ ] `ls -la ~/.claude/commands/lunch-break.md` shows a symlink pointing into your repo.

### 4.3 Functional checks (do these inside a cc session)

1. **Save path.** In a fresh cc session with a bit of real conversation, type `/lunch-break`. Expect:
   - A file appears under `~/.lunch-break/` with today's date and a kebab-slug.
   - Frontmatter includes `title`, `slug`, `saved_at`, `cwd`, and `branch` (if in a git repo).
   - Body contains all six sections: Goal, Done, Ongoing, Todo, Next, Notes.

2. **Dedup.** Type `/lunch-break` again in the same session. Expect the *previous* file with the same slug to be deleted and replaced — only one file per slug should remain.

3. **Aliases.** `/lunch`, `/break`, `/lunchbreak` all trigger the same save behavior.

4. **Restore — list mode.** Open a new cc session, type `/lunch-back`. Expect:
   - An `AskUserQuestion` prompt with up to 3 most-recent sessions.
   - Arrow keys move selection; cc shows "Other" as the last choice.
   - Picking one proceeds to load mode.

5. **Restore — load mode.** Confirm the loaded summary shows Goal / Done / Ongoing / Todo / Notes / **Next up** prompt and waits for confirmation.

6. **Attached filter.** After step 5, type `/lunch-back` again. Expect the just-loaded session to no longer appear in the list. Open its file on disk — expect a new `attached_at:` line in the frontmatter.

7. **Direct load.** Type `/lunch-back <slug>` for the attached session. Expect it to still load (explicit loads bypass the attached filter).

8. **Palette.** In cc, type `/` and scroll — all 7 commands appear without a `plugin-name:` prefix (because this ships as user-level commands, not a plugin).

### 4.4 Edge cases to spot-check

- [ ] `/lunch-back` when `~/.lunch-break/` is empty — cc prints the "no saved sessions" message and stops.
- [ ] `/lunch-back` when *all* sessions are attached — same no-saved-sessions message.
- [ ] `/lunch-break` in a trivial session (just "hi") — cc declines to save and tells the user.
- [ ] `/lunch-back foo` when `foo` matches nothing — cc says no match and stops.
- [ ] `/lunch-back foo` when `foo` matches multiple files — cc lists the candidates and stops (doesn't guess).

If any check fails, fix the corresponding file in §2 and re-verify. Don't ship with a failing check.

---

## 5. Hand off to the user

Once all checks in §4 pass, report back to the user with:

1. The repo path (and GitHub URL if published).
2. The two primary commands they should try first: `/lunch-break` and `/lunch-back`.
3. The full list of aliases.
4. A one-line reminder that handoffs live in `~/.lunch-break/` and that attached ones are filtered from the picker but still on disk.

That's it. Go build. Enjoy the coffee.
