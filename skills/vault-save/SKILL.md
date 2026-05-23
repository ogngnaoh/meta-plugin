---
name: vault-save
description: Use when the user invokes /vault-save (optionally with a subfolder name like 'claude-notes', 'claude-research-output', or 'meta-workflows'), OR when the user asks to save, export, copy, or stash the latest response into their Obsidian vault. Captures the most recent prior assistant response as a markdown file with YAML frontmatter (date, source=current git branch/worktree, tags) under the Obsidian vault at /Users/hoangngo/Documents/personal-vault/, fuzzy-matches the subfolder argument against existing folders, and prompts for confirmation before writing.
---

# Vault Save

Capture the most recent prior assistant response in this conversation and write it as a clean markdown note into the user's Obsidian vault at `/Users/hoangngo/Documents/personal-vault/`.

The user invokes this via `/vault-save <subfolder>`, where `<subfolder>` is a name or partial name like `claude-notes`, `notes`, `research`, `meta`, etc. The argument may also be empty — in which case you must ask.

## Why this skill exists

The user keeps an Obsidian vault for long-running threads of work (currently three top-level folders: `claude-notes`, `claude-research-output`, `meta-workflows`). They want to bookmark useful Claude Code outputs into the right thread without leaving the terminal and without losing provenance. So: infer cleanly, confirm cheaply, and never silently overwrite their notes.

## Workflow

Follow these steps in order. Use the user's existing vault conventions (lowercase-with-spaces filenames, `.md` extension, flat structure within each subfolder) — don't invent a new style.

### 1. Identify the content to save

Look back through this conversation and pick the **most recent substantial assistant response** prior to the `/vault-save` invocation. This is usually the message immediately before the user's slash-command invocation. If the prior response is trivial (one-line acknowledgement, a clarifying question, only tool calls with no prose) walk back further until you find a meaningful response.

If you genuinely cannot find substantive content to save — for example, the conversation only contains tool plumbing and questions — tell the user there's nothing meaningful to save and stop. Don't fabricate content.

### 2. Resolve the target subfolder

The vault root is `/Users/hoangngo/Documents/personal-vault/`. List its direct subdirectories with `ls` (skipping dotfiles like `.obsidian`). As of this skill's writing, the active subfolders are `claude-notes`, `claude-research-output`, and `meta-workflows`, but always re-list at runtime — the user may add or rename folders.

Match the user's argument against actual subfolders using this priority:

1. **Exact case-insensitive match** → use it.
2. **Unique substring match** (e.g., `research` → `claude-research-output`) → propose it, confirm before writing.
3. **Multiple substring matches** → ask the user to pick.
4. **No match** → ask the user: (a) pick from the existing list, (b) create a new subfolder with this name, or (c) cancel.
5. **No argument given** → list existing folders and ask which one.

Never auto-create a new folder without explicit user approval. New folder names should follow the existing convention (lowercase, hyphen-separated).

### 3. Infer a topic and propose a filename

Read the content from step 1 and infer a short topic phrase:

- 2-6 words, lowercase, **hyphens between words** (e.g., `lamin-vendor-metadata-status`, `silver-overview`)
- No leading articles (`the`, `a`)
- No trailing `.md` in the topic itself — you'll add the extension
- Concrete and searchable. Prefer specific nouns over generic verbs. `lamin-vendor-metadata-investigation` beats `database-notes` or `findings-summary`.

Compose the final filename as:

```
YYYY-MM-DD-<topic>.md
```

Example: `2026-05-11-lamin-vendor-metadata-status.md`

Note: the user's existing vault contains older notes that use spaces (e.g., `lamin vendor metadata status.md`). Do not rename those. New notes written by this skill use hyphens for consistency with the date prefix.

Use the current date (today's date as known by the environment). If the conversation references a specific date the user was working on something, prefer that — but only if it's unambiguous.

### 4. Check for collisions

Before proposing the write, check whether the target path already exists. If it does:

- Read the existing file
- Show the user a brief description: file path, last-modified date, first ~5 lines or so of existing content
- Ask: **overwrite** / **append** (separator + new content under the existing) / **new name** (let user supply one) / **cancel**

If the path is clear, proceed.

### 5. Confirm before writing

Always confirm with the user before writing. Show them:

- The full target path
- The inferred filename
- The frontmatter block you plan to use
- A one-line preview of the first non-frontmatter content

Use the `AskUserQuestion` tool when there's a real choice to make (ambiguous folder, collision, new-folder approval). For simple confirmation of the happy path, a short text proposal followed by `AskUserQuestion` with options like "looks good" / "edit filename" / "cancel" is fine.

### 6. Write the file

Before writing, resolve the `source` frontmatter value. The point of `source` is provenance — when the user finds this note in their vault months later, they should know which working context produced it. The current git branch (or worktree) is far more useful than a generic `claude-code` label, because branches map to specific lines of work.

Run this in the shell's working directory, in order, and use the first one that succeeds:

1. `git rev-parse --abbrev-ref HEAD` — current branch name. Worktrees return their own branch here, which is exactly what we want.
2. If that returns `HEAD` (detached state): `git rev-parse --short HEAD` for the short SHA.
3. If the directory is not a git repo at all: use the basename of the working directory (e.g., `basename "$PWD"`).

Use the resolved value as `source`. Don't fall back to `claude-code` — that label was a placeholder from before this skill knew about git context.

Once confirmed, write the file with this structure:

```markdown
---
date: YYYY-MM-DD
source: <resolved-branch-or-worktree>
tags: [<subfolder-name>]
---

<the content from step 1, reproduced faithfully as clean markdown>
```

Reproduce the content **faithfully**. Do not summarize, re-paragraph, or "improve" it. The user wants the actual response, not a redaction of it. The only transformations allowed:

- Strip any leading/trailing whitespace
- If the response contained Claude Code-specific UI artifacts (e.g., `<system-reminder>` blocks that leaked through, tool-call rendering), drop those — they're noise, not content.
- Convert any references to file paths that include the user's home dir to clean form if helpful, but don't be aggressive.

### 7. Report back

After writing, report:

- The exact path written
- The filename
- A one-line note of any choices you made on the user's behalf (e.g., "created new subfolder `protein-engineering`", "inferred topic from your second-most-recent response since the latest was a clarifying question")

Keep this terse — one short paragraph or three bullets max.

## Frontmatter reference

The frontmatter is intentionally minimal so it stays out of the way in Obsidian. Tags is a single-item list containing only the subfolder name. The user can enrich tags inside Obsidian later. `source` is the git branch (or worktree, or working-directory basename if not in a repo) resolved at write time — see step 6.

```yaml
---
date: 2026-05-15
source: hoang-branch
tags: [claude-notes]
---
```

## Examples

**Example 1 — happy path:**
- User just got a long analysis of an interesting Claude Code workflow.
- User runs `/vault-save notes`.
- You fuzzy-match `notes` → `claude-notes`, infer topic `subagent-dispatch-patterns`, propose path `/Users/hoangngo/Documents/personal-vault/claude-notes/2026-05-15-subagent-dispatch-patterns.md`, show the frontmatter and ask confirm.
- User confirms. You write the file and report the path.

**Example 2 — ambiguous arg:**
- User runs `/vault-save meta`.
- `meta` is a substring of `meta-workflows` only (one match), so propose that. Confirm. Write.
- If there were also a `metabolomics` folder, you'd list both and ask which one.

**Example 3 — new folder:**
- User runs `/vault-save crispr-screens`.
- No existing folder matches. Ask: "No subfolder matches `crispr-screens`. Options: create new folder `crispr-screens`, pick from existing (claude-notes / claude-research-output / meta-workflows), or cancel?"
- User says create. You `mkdir` the folder, then continue to step 3.

**Example 4 — collision:**
- Target file already exists at `claude-notes/2026-05-15-subagent-dispatch-patterns.md`.
- Show the user the existing file's first lines and last-modified date.
- Ask: overwrite / append / new name / cancel.
- If append, separate with `\n\n---\n\n## Appended <timestamp>\n\n` before the new content.

**Example 5 — no arg:**
- User runs `/vault-save` with no argument.
- List the existing subfolders and ask which one. Don't guess from the content alone — the user may want to file something in `meta-workflows` even if it sounds research-y.

## Anti-patterns

- **Don't summarize the response.** The user wants the verbatim content, not a TL;DR.
- **Don't create folders without permission.** Even if the new name looks reasonable.
- **Don't overwrite without showing the diff.** Notes lost this way are unrecoverable.
- **Don't pick a generic filename** like `notes.md` or `output.md`. The point of an Obsidian vault is searchability; vague filenames defeat that.
- **Don't skip confirmation just because the inputs are clean.** A 2-second confirm is much cheaper than a misplaced or misnamed note.
- **Don't read from the system clipboard.** The skill works off the conversation context, not `pbpaste`. The user can edit the inferred filename/topic during confirmation if they want a different framing.
