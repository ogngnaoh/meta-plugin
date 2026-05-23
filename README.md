# meta-plugin

Meta-skills for **deliberate orchestration of installed skills** — consult them as advisors, dispatch them across parallel subagents, or run them as procedural guides. Plus `vault-save`, a personal-utility skill for bookmarking useful Claude responses into an Obsidian vault without leaving the terminal.

The orchestration trio (`skill-consult`, `skill-dispatch`, `skill-run`) don't do work themselves; they coordinate the *other* skills you have installed so their domain expertise actually shapes how Claude answers and executes. `vault-save` is the odd one out — it doesn't orchestrate anything; it captures and persists conversation output.

> **Prerequisite (orchestration trio):** the three meta-skills get their value from the skills they orchestrate. Install at least a few domain or process skill packs (e.g. `superpowers`, `feature-dev`, `plugin-dev`, language/framework packs) first — otherwise there's nothing to consult, run, or dispatch. `skill-dispatch` specifically requires `superpowers:dispatching-parallel-agents`.

## Components

| Name | Type | Purpose |
|------|------|---------|
| `skill-consult` | Skill | Pools 3–5 relevant skills as consultants and synthesizes an answer to a question or trade-off |
| `skill-dispatch` | Skill | Decomposes multi-domain work into 2–6 independent subtasks, each run by a parallel subagent loaded with its own skills |
| `skill-run` | Skill | Loads 2–5 skills as procedural guides and executes a task under their active workflow |
| `vault-save` | Skill | Captures the most recent assistant response and writes it as a dated markdown note (with git-branch provenance) into an Obsidian vault |

## When to use which skill

```
Question / trade-off / "what should I…"   →  skill-consult
Single-domain task to actually DO         →  skill-run
Multi-domain / parallel investigation     →  skill-dispatch
Bookmark this response into the vault     →  vault-save
```

A quick mental model:

- **consult** — skills are *advisors*. You ask, they weigh in, you synthesize the answer.
- **run** — skills are *procedures*. You load them, then execute the task by following their workflows.
- **dispatch** — skills are *equipment for parallel workers*. You fan out, each subagent loads its own subset, you synthesize the returns.
- **vault-save** — not an orchestrator. A small utility that writes the last assistant response into your Obsidian vault as a dated markdown note.

## Installation

**1. Register the marketplace** in `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "meta-plugin": {
      "source": {
        "source": "github",
        "repo": "ogngnaoh/meta-plugin"
      }
    }
  }
}
```

**2. Install the plugin:**

```
/plugin install meta-plugin
```

**3. Verify** with `/plugin list` — you should see `meta-plugin` listed.

## Skills Reference

### `skill-consult`

Use when you have a **question, decision, or trade-off** and you want pooled domain expertise rather than generic reasoning.

**Trigger phrases / patterns:**
- `/skill-consult <question>`
- "interview me on X", "help me think through Y"
- "X vs Y?", "what should I pick for…", architectural decisions, library/tool choices, debugging strategies

**What it does:** Picks 3–5 genuinely relevant installed skills, loads them as *consultants* (not executors), and synthesizes an answer. In **interview mode** (when phrasing implies a decision), it asks MCQ-style clarifying questions citing which skill informs each option, capped at 2 rounds. Ends with a `Consulted:` footer naming the skills used.

**Example:**
```
/skill-consult should we use Polars or DuckDB for a 200M-row analytics workload?
```

**Don't use for** actionable tasks (use `skill-run`) or multi-domain parallel work (use `skill-dispatch`).

---

### `skill-run`

Use when you want a **task actually done** and specialized skill workflows should shape execution.

**Trigger phrases / patterns:**
- `/skill-run <task>`
- "implement…", "refactor…", "plan a feature for…", "review this PR…"
- Any procedural task where a skill's prescribed workflow beats general defaults.

**What it does:** Picks 2–5 skills whose workflows actively change *how* the work is done — process skills first (planning, TDD, debugging), domain skills second. Loads them, follows their procedures, honors their checkpoints (e.g. plan-approval, brainstorming sign-off, failing-test gates). Ends with an `Applied:` footer naming the skills used.

**Example:**
```
/skill-run plan and implement a rate-limiting middleware for the FastAPI service
```

**Don't use for** pure questions (use `skill-consult`) or multi-domain parallel work (use `skill-dispatch`).

---

### `skill-dispatch`

Use when work spans **multiple independent domains or lenses** and parallelism makes sense.

**Trigger phrases / patterns:**
- `/skill-dispatch <task>`
- "audit X for security + perf + tests"
- "compare libraries A, B, C"
- "plan backend + frontend + data pipeline"
- Anything described as "in parallel".

**What it does:** Decomposes the task into 2–6 independent subtasks, picks 1–4 skills per subtask, then delegates the actual concurrent dispatch to `superpowers:dispatching-parallel-agents`. Each subagent runs with its own skills loaded. When all return, it synthesizes — surfacing cross-cutting issues, contradictions, gaps, and priorities. Ends with a `Dispatched:` footer mapping subtasks to skills.

**Example:**
```
/skill-dispatch audit src/api/ for security, performance, and test coverage in parallel
```

**Don't use for** single-domain work, sequentially dependent steps, or pure questions.

**Dependency:** dispatch leans on `superpowers:dispatching-parallel-agents` for the spawn mechanics. If that skill isn't installed, dispatch will tell you and stop rather than silently fall back.

---

### `vault-save`

Use when you want to **bookmark a Claude response into a long-running knowledge base** (an Obsidian vault on disk) without leaving the terminal.

**Trigger phrases / patterns:**
- `/vault-save <subfolder>` (e.g. `/vault-save notes`, `/vault-save research`, `/vault-save meta`)
- "save this to my vault", "stash that in claude-notes", "export this response"

**What it does:** Grabs the most recent substantial assistant response in the conversation, infers a short hyphenated topic, and writes it to `<vault>/<subfolder>/YYYY-MM-DD-<topic>.md` with this frontmatter:

```yaml
---
date: 2026-05-23
source: <git-branch-or-worktree-or-cwd-basename>
tags: [<subfolder>]
---
```

The `source` field gives provenance — it's resolved at write time from `git rev-parse --abbrev-ref HEAD` (falling back to the short SHA for detached HEAD, then to the working-directory basename if you're not in a git repo). The body is the response reproduced faithfully, not summarized.

Fuzzy-matches the subfolder argument against existing folders (e.g. `research` → `claude-research-output`). Confirms before writing, asks before creating a new folder, and never silently overwrites — if the target path exists you'll be offered overwrite / append / new-name / cancel.

**Example:**
```
/vault-save research
```
→ resolves `research` to `claude-research-output/`, infers a topic from the prior response, shows you the proposed path + frontmatter, writes on confirm.

**Heads-up for forks:** the skill assumes the author's vault layout — `/Users/hoangngo/Documents/personal-vault/` with subfolders like `claude-notes`, `claude-research-output`, `meta-workflows`. If you fork this plugin, edit the vault root and the subfolder names referenced in `skills/vault-save/SKILL.md` before using it. No env-var/config knob yet; it's a personal utility.

**Don't use for** orchestrating other skills — that's what the trio is for. `vault-save` is just a persistence helper.

## Updating

```
/plugin update meta-plugin
```

## Development

```bash
git clone https://github.com/ogngnaoh/meta-plugin
cd meta-plugin
```

- Skills live in `skills/<skill-name>/SKILL.md`

Reload Claude Code after editing.
