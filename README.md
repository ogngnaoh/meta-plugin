# meta-plugin

A small collection of meta-skills for Claude Code. `agent-team` interviews you to design a role-based agent team that fits your task and runs it to produce the work product — with single-responsibility teammates backed by real subagent definitions that enforce per-role tool whitelists. `doc-system` generates `PRD.md` / `SPEC.md` / `CLAUDE.md` via an interview-and-quality-gate workflow. `system-explain` breaks any architecture or system design down to first-principles concepts. `ship-slice` runs the slice-ship ritual — flipping status in `milestone.md`, advancing the "Next session start here" pointer in `handoff.md`, and shutting down the slice's teammates/goals, all in one atomic commit. `vault-save` bookmarks useful Claude responses into an Obsidian vault without leaving the terminal.

## Components

| Name | Type | Purpose |
|------|------|---------|
| `agent-team` | Skill | Interviews you to design a role-based agent team that fits a task and runs it to produce the work product. Roles Scout/Builder/Critic/Evaluator/Integrator, backed by real subagent definitions with per-role tool whitelists |
| `ship-slice` | Skill | Atomically ships a vertical slice — flips status in `milestone.md` (the single live home of status), advances handoff's "Next session start here" pointer, shuts down the slice's teammates/standing goals, and lands it in one commit on the main-flow branch |
| `doc-system` | Skill | Generates `PRD.md` / `SPEC.md` / `CLAUDE.md` for a Claude-Code-driven project via interview, optional brownfield inventory, and a pre-flight quality gate |
| `system-explain` | Skill | Explains the foundational system-design concepts behind a pasted architecture, blog, or code from first principles — concept map, causal deep-dives, binding-constraint synthesis |
| `vault-save` | Skill | Captures the most recent assistant response and writes it as a dated markdown note (with git-branch provenance) into an Obsidian vault |

## When to use which skill

```
Build & run a role-based agent team        →  agent-team
Mark a finished vertical slice shipped     →  ship-slice
Generate PRD / SPEC / CLAUDE.md docs       →  doc-system
"Explain the concepts behind this design"  →  system-explain
Bookmark this response into the vault      →  vault-save
```

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

### `agent-team`

Invoke it when you want a role-based agent team on a task — you drive as lead. The skill does one thing, in two steps: **interview** you to design the team that fits, then **build** it — scaffold and run the teammates to produce the work product.

**Trigger phrases / patterns:**
- `/agent-team <task>`
- "spin up a team for…", "orchestrate agents to…", "build this with a multi-agent team"

**What it does:** First it **interviews** you to establish the task, design the team it needs (which roles, who owns what, how they coordinate — drawing on common patterns like adversarial multi-lens review, competing-hypothesis debugging, or a live-negotiated parallel build), and pin explicit acceptance criteria. Then it **builds**: scaffolds single-responsibility teammates — Scout/investigator, Builder, Integrator, and (when the team's design calls for verification) review roles Critic + Evaluator — each backed by a real subagent definition (a shipped `agents/agent-team-*` or a better-matching installed agent) that enforces a tool whitelist, spawned with a full 4-part contract; the lead orchestrates and synthesizes but **never authors the work product**, works teammates under strict file ownership, and caps the loop.

**Ships 5 role definitions** under `agents/`: `agent-team-{scout, builder, critic, evaluator, integrator}`.

**Example:**
```
/agent-team build the real-time notifications feature end-to-end
```

**Don't use for** a single quick delegation, a pure question, or a trivial one-file edit — those don't need a team. Note that a team's documented cost is ~7× a standard session when teammates run in plan mode (~linear per teammate otherwise), scaling with team size.

---

### `ship-slice`

Use when a **vertical slice is finished and ready to mark shipped** — the most error-prone bookkeeping moment in the milestone/slice workflow, where doing it by hand at the end of a long session tends to leave the docs half-updated.

**Trigger phrases / patterns:**
- `/ship-slice`
- "ship this slice", "mark slice N shipped", "close out the slice", "finish the last slice of the milestone"

**What it does:** Runs the slice-ship ritual as one atomic unit so the project's doc state can never end up half-updated. It confirms the project uses the convention (`docs/milestones.md` exists), identifies the `in-progress` slice (asking if ambiguous), checks the work is actually verified (and flags when the verification came entirely from the span that wrote the code), and does worktree/branch safety so the commit can't vanish on a throwaway branch. Then it applies the two doc edits together — flips the slice's status in `milestone.md` (the only home of slice status) and overwrites `handoff.md` (three sections, ≤40 lines: next-session pointer, current state, active concerns) — shuts down the slice's teammates/standing goals/spent worktrees, and lands the doc edits in **one commit on the main-flow branch**. If it's the milestone's last slice, the milestone-close (`milestones.md` → shipped, next `← active`) rides in the same commit by default. User-invoked only (`disable-model-invocation: true`).

**Example:**
```
/ship-slice
```

**Heads-up for forks:** assumes the author's milestone/slice/handoff convention — a `docs/milestones.md` index and per-milestone `docs/NN-milestone/` folders with `milestone.md` + `handoff.md`. The skill itself is the single home of the ship ritual's step list (the author's global `CLAUDE.md` points here). If your project doesn't use that layout, the skill stops at its first check rather than inventing structure.

**Don't use for** mid-implementation work (finish and verify the slice first — this is the bookkeeping+commit step), projects that don't use the milestone/slice docs, or *starting* the next slice (shipping ≠ starting; it stops after the report).

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

## Content generation

### `doc-system`

Use when you need to **generate or refresh the canonical project docs** that a Claude-Code-driven solo project leans on: `PRD.md` (what + why, tech-agnostic), `SPEC.md` (architecture + alternatives + cross-cutting concerns + rollout), and `CLAUDE.md` (commands, stack, conventions, gotchas for Claude Code sessions). One, two, or all three on request.

**Trigger phrases / patterns:**
- "write me a CLAUDE.md", "I need a PRD and SPEC for this feature"
- "set up docs for my new project", "draft a PRD for X"
- Any request to produce/refresh PRD / SPEC / CLAUDE.md for a repo.

**What it does:** Detects which of the three docs you want, then gathers context based on how you invoked it — greenfield (no inventory), brownfield (dispatches a general-purpose subagent to inventory the repo first), or pointed-at-files (reads only what you handed it). Interviews to fill remaining gaps using `AskUserQuestion`-style options where the answer space is small. Drafts each requested doc, then runs a hard pre-flight quality gate (`references/quality-gate.md`) — failing checks become explicit `[NEEDS CLARIFICATION: ...]` markers rather than silent omissions. Emits to `./PRD.md` / `./SPEC.md` / `./CLAUDE.md` at repo root. Never overwrites an existing doc without confirmation.

**Example:**
```
set up PRD, SPEC, and CLAUDE.md for this new analytics service
```

**Don't use for** session-volatile artifacts (`handoff.md`, `plan.md`, `tasks.md`, `milestone.md`, files under `agent_docs/`) — those are synthesized mid-flight by Claude Code, not produced by this skill. The skill embeds *conventions* for these inside CLAUDE.md so Claude Code shapes them well when the time comes, but doesn't pre-create them. Also not a substitute for ADRs; architectural rationale lives in SPEC's mandatory "Alternatives Considered" section.

---

### `system-explain`

Use when you've **pasted an architecture, system design, tech blog, or code** and want the foundational concepts behind it explained from first principles — for your own learning, not for execution.

**Trigger phrases / patterns:**
- "explain the concepts behind this", "break this design down"
- "help me understand why this works", "what's actually going on here"

**What it does:** Detects the system-design concepts the input touches (`CONCEPTS DETECTED` map), then deep-dives each with a fixed causal template — what it is, why it *must* exist (derived from physics, economics, or impossibility results like CAP/PACELC/FLP), the trade-off it costs, how *this* input uses it, and what breaks at 10×/100×. Closes by naming the single **binding constraint** the whole design is downstream of. Backed by a bundled first-principles framework (`references/system-design-framework.md`); supplements with broader knowledge and web search where it helps. Pure explainer — it never quizzes or drills.

**Example:**
```
explain the concepts behind this: Cassandra + write-through Redis cache + consistent hashing, multi-region
```

**Don't use for** doing design work or making decisions — it teaches the concepts in something you hand it, not what to pick.

## Updating

```
/plugin update meta-plugin
```

## Development

```bash
git clone https://github.com/ogngnaoh/meta-plugin
cd meta-plugin
```

- Skills live in `skills/<skill-name>/SKILL.md`, with an optional `references/` subfolder for templates and longer-form material loaded on demand (see `skills/doc-system/`)

Reload Claude Code after editing.
