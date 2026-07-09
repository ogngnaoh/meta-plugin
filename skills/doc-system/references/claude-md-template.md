# CLAUDE.md — Canonical Template (full)

This is the long form of the CLAUDE.md template. The SKILL.md body has the abridged sketch; this file is the source of truth.

CLAUDE.md is different in kind from PRD and SPEC: it's read by a machine at the start of every session, not by humans during reviews. Its primary failure mode is therefore not "confusing readers" but **"consuming the context window and crowding out actual code."** Treat it like code, not like documentation.

## Anthropic's include / exclude lists (the load-bearing rules)

From Anthropic's official "Best practices for Claude Code":

**✅ Include:**
- Bash commands Claude can't guess (build, test, lint, dev server)
- Code style rules that **differ from defaults** for the language/framework
- Testing instructions (single-test invocation, fixture conventions)
- Repository etiquette (branch names, commit conventions, PR rules)
- Project-specific architectural anchors (not paths that move)
- Developer-environment quirks
- Common gotchas — tribal knowledge a senior engineer would tell a new hire

**❌ Exclude:**
- Anything Claude can figure out by reading code
- Standard language conventions Claude already knows
- Detailed API documentation (link instead)
- Information that changes frequently (current sprint, who's on call)
- Long explanations or tutorials
- File-by-file descriptions of the codebase
- Self-evident practices like "write clean code"
- **Secrets, API keys, connection strings** (never. ever.)
- Rules a linter or `.editorconfig` already enforces — "if a violation of the rule would block a merge in CI, the rule belongs in CI"

Anthropic's own warning: **"Bloated CLAUDE.md files cause Claude to ignore your actual instructions."**

---

## Section structure (load-bearing)

Section order is deliberate. Claude attends most reliably to the **start and end** of the file (primacy and recency bias). The order below puts the most behavior-shaping content where attention is highest.

1. **Pointers** — short list of `@`-references to the other docs (SPEC, handoff, ADRs if any). Without this, Claude never reads SPEC.md.
2. **Commands** — build, test (all + single), typecheck, lint, dev server. The single highest-leverage section; most failure modes come from this being missing or wrong.
3. **Stack** — language + version, framework + version, database, key libraries with versions that matter. "React" without "React 19" leaves Claude free to pick stale patterns.
4. **Project structure** — directories with one-phrase purposes (5 words max). High-churn paths don't belong here; stable architectural anchors do.
5. **Critical constraints** — `NEVER` / `ALWAYS` lines, reserved for danger and invariants. Use sparingly; overuse devalues the signal.
6. **Conventions (deviations from defaults only)** — naming, error handling, import ordering — only when non-standard.
7. **Gotchas** — non-obvious behaviors that have already bitten. Each one is a debugging session you don't want to repeat.
8. **Artifact conventions block (optional, at the end)** — for projects following the user's own solo-dev workflow (handoff.md / plan.md / tasks.md / milestone.md synthesized mid-session by Claude Code). Recency bias means it gets attended to. Skip if the project doesn't use this workflow.

---

## Reusable CLAUDE.md template

```markdown
# CLAUDE.md

## Pointers
- @PRD.md — what we're building and why
- @SPEC.md — how it's built; architecture, alternatives, rollout
- @docs/handoff.md — read at session start (overwritten each session)
- (Add other authoritative project docs here as @-pointers)

## Commands
- Install: `<command>`
- Dev server: `<command>`
- Build: `<command>`
- Test all: `<command>`
- Test single: `<command pattern>` — prefer this; full suite is slow
- Typecheck: `<command>`
- Lint: `<command>`
- Format: `<command>`

## Stack
- <Language + version, framework + version>
- <Database + version, ORM>
- <Key libraries with versions that matter>

## Project Structure
- src/<dir>/ — <purpose, 5 words max>
- src/<dir>/ — <purpose>
- docs/ — project documentation
- tests/ — <what shape, mirrors src/?>

## Critical Constraints
- NEVER <genuinely dangerous operation>
- ALWAYS <critical pattern, e.g. "use parameterized queries">
- <Security boundary, performance floor, or invariant>

## Conventions (deviations from defaults only)
- <Naming convention if non-standard>
- <Error handling pattern>
- <Import ordering rule if non-standard>
- <Test file location>

## Gotchas
- <Non-obvious behavior that has bitten before>
- <Editor / build / cache quirk Claude wouldn't infer>

## Artifact conventions  (optional — only if your workflow uses these)

These files are synthesized on the fly by Claude Code during a session.
Do NOT pre-create them.

### handoff.md (overwrite each session — never append — ≤40 lines, 10-15 typical)
Three sections, nothing else:
- Next session start here: <literal first action>
- Current state: <branch, what's deployed, what's mid-flight>
- Active concerns: <fragile areas, open questions>
No status tables, no commit refs, no ship log — milestone.md and git hold those.

### plan.md (per feature or milestone, 1-2 pages max)
Sections: Goal (one sentence) / Components (with file paths) /
Data model changes / Integration points / Testing strategy /
Exit criteria (testable) / Non-goals.

### tasks.md (5-20 items, paired with plan.md)
Markdown checklist. Each task: file paths touched, verification step,
atomic commit, sized 15-45 minutes. Order so every task leaves main green.
For bug fixes, the first task is always the failing test.

### milestones.md (root of docs/ if you use milestone-based work)
Numbered one-liners in execution order, with the active one marked `← active`.
```

---

## Length norms

| Source | Recommended length |
|---|---|
| Anthropic official memory docs | "Under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence." |
| HumanLayer analysis | "As few instructions as possible" (Claude Code's harness already uses ~50 of the ~150–200 instruction budget) |
| codewithmukesh | 50–100 lines in the root file, imports for detail |
| TurboDocx | ~200 lines target, ~300 ceiling before adherence drops |

**The skill's rule of thumb:** target under 150 lines for the root CLAUDE.md. Over 200 is a warning, not a block — but every line over should pass Anthropic's line-by-line test:

> "Would removing this cause Claude to make a mistake?"

If the answer is no, cut. If you keep failing the test, extract to `agent_docs/<name>.md` and reference with `@agent_docs/<name>.md`.

---

## Imperative tone

Per Anthropic's guidance, instructions should be **direct commands**, not observations:

| Don't write | Write |
|---|---|
| "We generally try to avoid inline mocks." | "Avoid inline mocks; use the shared fixture in `tests/fixtures/`." |
| "It might be a good idea to typecheck." | "Run `pnpm typecheck` before declaring a task done." |
| "Be careful with the database." | "Always use parameterized queries; never interpolate user input into SQL." |

Reserve `IMPORTANT` / `YOU MUST` for the one or two rules that are genuinely critical. Overuse devalues the signal — when everything is important, nothing is.

---

## Anti-patterns specific to CLAUDE.md

These are the recurring failure modes. The quality gate explicitly checks for several of these.

| Anti-pattern | Symptom | Fix |
|---|---|---|
| Auto-generated boilerplate prose | Mirrors README structure, states obvious things, "Welcome to…" or "This is a TypeScript project that…" | Delete; hand-write from this template. ETH Zurich (2026) measured auto-generated CLAUDE.md files *reduce* agent success 2-3% vs hand-written. |
| Instruction overload | >200 lines or >150 discrete instructions | Extract to `agent_docs/`, reference with `@`. |
| Negative framing dominance | Lists of DON'Ts without DOs | Rewrite each as positive instruction. Reserve NEVER for danger. |
| Stale "Current Focus" section | Describes work completed weeks ago | Delete — that's what `handoff.md` is for. |
| Implementation details embedded | File paths beyond top-level, function names, line numbers | Move to `agent_docs/`. CLAUDE.md describes patterns, not specifics. |
| Style rules a linter already enforces | Rules duplicated in two places drift | Configure the linter; remove from CLAUDE.md. |
| Secrets / API keys / connection strings | Anything sensitive in the file | NEVER. Move to env vars and reference by env-var name only. |
| Suggestion-form instructions | "We generally try to …" | Rewrite as imperative. |

---

## How this lands

1. Write CLAUDE.md at the repo root.
2. Do **not** commit auto-generated boilerplate — hand-write from this template instead.
3. For personal project quirks that shouldn't be foisted on collaborators, use `CLAUDE.local.md` (gitignored).
4. Review weekly: apply the line-by-line test to anything added that week. Prune ruthlessly.
5. When Claude makes a first-occurrence mistake, add a one-line fix to CLAUDE.md before the lesson evaporates. When it makes the same mistake a third time despite CLAUDE.md saying not to, the file is too long — extract older content to `agent_docs/`.
