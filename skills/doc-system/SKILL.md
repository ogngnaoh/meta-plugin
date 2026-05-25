---
name: doc-system
description: >
  Manually-invoked skill for generating PRD.md (what + why,
  tech-agnostic), SPEC.md (architecture + alternatives + rollout +
  cross-cutting concerns), and CLAUDE.md (commands, stack, conventions,
  gotchas for Claude Code sessions). Interviews to fill gaps, dispatches
  an inventory subagent on brownfield repos, applies a hard pre-flight
  quality gate per doc, emits one, two, or all three on request.
---

# Documentation System for Solo Developers + Claude Code

This skill produces **three documents** for a Claude-Code-driven project:

| Artifact | Purpose | Volatility |
|---|---|---|
| **PRD.md** | What we're building and why, for whom, and how we'll know it worked. Technology-agnostic. | Slow (monthly–quarterly) |
| **SPEC.md** | How the system is built: architecture, data model, alternatives considered, cross-cutting concerns, rollout. | Per-feature (weekly–monthly) |
| **CLAUDE.md** | What Claude Code needs every session: commands, stack, conventions, gotchas. | Mistake-driven |

The skill operates in a **single Generate mode**: interview → fill gaps → apply quality gate → emit whichever of the three docs the user asked for (one, two, or all three).

## What this skill does NOT produce

Session-volatile artifacts that Claude Code synthesizes mid-flight: `handoff.md`, `plan.md`, `tasks.md`, `milestone.md`, `features.md`, files under `agent_docs/`. These change too fast or depend on real codebase context that doesn't exist during chat-side scoping. The skill embeds **conventions** for these inside CLAUDE.md (see the optional "Artifact conventions" block in `references/claude-md-template.md`) so Claude Code shapes them well when the time comes — but it does not pre-create them.

It also does not produce ADRs. Architectural rationale lives in SPEC's mandatory **Alternatives Considered** section instead. If your project genuinely outgrows that and needs separate ADRs, that's a downstream concern beyond this skill's scope.

---

## Core Workflow

For every invocation, follow these steps in order:

1. **Detect scope.** Parse the user's request: which of the three docs do they want? Possibilities: one (e.g., "write me a CLAUDE.md"), two (e.g., "I need a PRD and SPEC for this feature"), all three (e.g., "set up docs for my new project"). If genuinely ambiguous, ask once.
2. **Gather context.** Choose the path that fits how the user invoked you:
   - **User pointed at specific files** (e.g., "read @docs/architecture.md and write a SPEC"): Read those files inline. Treat them as the context the user is handing you; do not also inventory the rest of the repo. The full interview and gate workflow still applies.
   - **Brownfield (any files present in the repo) and no pointed file**: Dispatch a general-purpose subagent to inventory before interviewing. Use the [Brownfield Inventory Subagent](#brownfield-inventory-subagent) template below. Use the returned summary to skip interview questions the inventory already answers.
   - **Greenfield (empty repo)**: No inventory step. Go straight to scope detection and interview.

   In all three cases: if PRD/SPEC/CLAUDE.md already exist (whether pointed-at or surfaced by the subagent), **read them as interview context, not as something to regenerate**. Honor any global conventions in `~/.claude/CLAUDE.md` (e.g., if the user's workflow uses `docs/milestones.md` / handoff.md, point the generated CLAUDE.md at those).
3. **Track multi-doc work.** If the user requested **2+ docs** (any combination of PRD / SPEC / CLAUDE.md), immediately create a TodoWrite list with milestones for each requested doc plus interview and landing: (1) interview, (2) PRD draft + gate (if requested), (3) SPEC draft + gate (if requested), (4) CLAUDE.md draft + gate (if requested), (5) land. Mark each completed as you finish it. Single-doc requests skip TodoWrite — the overhead isn't worth it.
4. **Interview to fill gaps.** Ask only what's missing. Pick from the questions in [The Interview Protocol](#the-interview-protocol) below; cover the sections relevant to whichever docs you're producing. One substantive question at a time when possible; prefer `AskUserQuestion`-style options over open-ended prompts when the answer space is small. If the user shared planning work from a prior session, honor those decisions and don't re-ask settled questions.
5. **For each requested doc, in this order: PRD → SPEC → CLAUDE.md.**
   - Draft the doc using the inline template below, expanding from `references/<doc>-template.md` as needed.
   - Run the pre-flight quality gate (`references/quality-gate.md`).
   - For each check that fails: either ask the user to resolve, or insert a `[NEEDS CLARIFICATION: ...]` marker at the point of impact and let the gate pass on that explicit deferral.
   - Tell the user the gate verdict ("PRD passed gate with 1 [NEEDS CLARIFICATION] marker") before emitting.
   - Emit the doc to its canonical path: `./PRD.md`, `./SPEC.md`, or `./CLAUDE.md` at repo root. Never overwrite an existing doc without confirmation.
6. **Tell the user how to land the docs.** Canonical paths, commit ordering, and any caveats specific to their workflow (e.g., "your global CLAUDE.md uses `docs/NN-milestone/` — leave room in CLAUDE.md to point at that").

When the user requests multiple docs in one session, also apply the cross-doc consistency checks at the end of `quality-gate.md` before finalizing.

---

## The Interview Protocol

Don't run these as a quiz. Pick what's missing, ask conversationally. Cover only what the docs you're producing need.

**Project identity (PRD + SPEC + CLAUDE.md):**
- What does this project do? (one sentence)
- Greenfield or existing?

**Target user & goals (PRD only):**
- Who is this for? Even if just you, *which version of you* — be specific.
- What problem does it solve, in one paragraph?
- What does success look like? Force a numeric, time-bounded metric.
- What is this project NOT trying to be? (Need ≥3 non-goals.)
- What are the 1–3 most important user stories? For each, draft Given/When/Then acceptance scenarios.

**Tech stack (SPEC + CLAUDE.md, never PRD):**
- Language + version, framework + version, database, key libraries with versions that matter.
- Build / test / typecheck / lint commands. Single-test invocation.
- Deploy target.

**Architecture & alternatives (SPEC only):**
- High-level architecture: what are the boxes, what are the arrows?
- Data model: key entities and their relationships.
- What's the obvious alternative design? Why did you reject it?
- What other alternative did you consider? Why rejected?
- What's the "do nothing" baseline — what happens if you ship nothing?

**Cross-cutting concerns (SPEC only):**
- Security: auth model, secrets handling, blast radius if compromised.
- Privacy: PII handling, retention, deletion.
- Observability: logs, metrics, traces, SLOs.

**Existing docs / pain signals (CLAUDE.md only):**
- What docs exist now? When were they last updated?
- Has Claude Code been making repeated mistakes? What kind? (These become CLAUDE.md gotchas.)
- Are you re-explaining the same context across sessions?

---

## Pre-Flight Quality Gate (summary)

The full gate is in `references/quality-gate.md`. Apply it before emitting *any* doc. **It's a hard gate** — surface failures to the user before emitting; either resolve the gap or mark it `[NEEDS CLARIFICATION: ...]` at the point of impact.

**Highest-leverage checks (each is enforced):**

**PRD gate (4 most important):**
- ≥3 non-goals
- ≥1 measurable, time-bounded goal
- ≥1 FR using `FR-NNN` numbering with "System MUST …" form
- **Zero technology mentions** (frameworks, libraries, databases — those belong in SPEC)

**SPEC gate (4 most important):**
- Alternatives Considered with ≥2 alternatives + "do nothing" baseline
- Security + Privacy + Observability each addressed at minimum
- ≥1 numeric goal
- Testing & validation section non-empty

**CLAUDE.md gate (4 most important):**
- **No secrets, API keys, or connection strings** (always a hard block)
- Build / test / typecheck / lint commands present
- No file paths beyond top-level directory anchors
- No auto-`/init`-style boilerplate prose

---

## Inline Templates

Abridged sketches. Full canonical templates with filled examples and per-section rationale live in `references/`.

### PRD.md (sketch)

```markdown
# PRD: <Name>

**Status:** Draft | In Progress | Approved
**Last updated:** YYYY-MM-DD

## 1. Problem
<One paragraph: who has this pain, when, why current solutions fail.>

## 2. Target users
- **Primary:** <persona, context>

## 3. Goals & success metrics
- **G1:** <verb + outcome> — **Metric:** <numeric, time-bounded>

## 4. Non-goals
- N1: ...   N2: ...   N3: ...

## 5. User stories
### Story 1 — <Title>
**As a** <persona> **I want** <capability> **so that** <benefit>.
**Acceptance scenarios:**
- Given <state>, when <action>, then <outcome>.

## 6. Functional requirements
- **FR-001:** System MUST <capability>.
- **FR-002:** System MUST <capability> [NEEDS CLARIFICATION: <gap>].

## 7. Non-functional requirements
## 8. Edge cases
## 9. Assumptions & dependencies
## 10. Open questions
```

→ Full template, filled example, and discipline notes: `references/prd-template.md`.

### SPEC.md (sketch)

```markdown
# SPEC: <Name>

**Status:** Draft | In Review | Approved | Implemented
**Last updated:** YYYY-MM-DD
**Related PRD:** ./PRD.md

## 1. TL;DR
<3–5 sentences: what's built, chosen approach, single biggest trade-off.>

## 2. Context
## 3. Goals  (≥1 numeric)
## 4. Non-goals
## 5. Technical context
<language/version, deps, storage, testing, target, performance, scale>

## 6. Proposed design
### 6.1 Architecture overview
### 6.2 Data model
### 6.3 Interfaces / contracts
### 6.4 Key flows

## 7. Alternatives considered  (≥2 alternatives + do-nothing baseline — required)
### Alt A: <Name>
### Alt B: <Name>
### "Do nothing" baseline

## 8. Cross-cutting concerns  (security + privacy + observability all required)
## 9. Rollout & migration
## 10. Testing & validation
## 11. Risks & open questions
```

→ Full template, filled example, and discipline notes: `references/spec-template.md`.

### CLAUDE.md (sketch)

```markdown
# CLAUDE.md

## Pointers
- @PRD.md
- @SPEC.md
- @docs/handoff.md  (if your workflow uses it)

## Commands
- Install / Dev / Build / Test all / Test single / Typecheck / Lint / Format

## Stack
- <Language + version, framework + version, DB, key libs with versions>

## Project Structure
- src/<dir>/ — <5-word purpose>
- (top-level only; no file paths beyond directory anchors)

## Critical Constraints
- NEVER <genuinely dangerous>
- ALWAYS <critical pattern>

## Conventions (deviations from defaults only)
- (rules a linter can enforce don't belong here)

## Gotchas
- <Non-obvious behavior that has bitten>

## Artifact conventions  (optional — for workflows that use handoff.md / plan.md / tasks.md)
<see references/claude-md-template.md for the block>
```

→ Full template, Anthropic include/exclude lists, length norms, and discipline notes: `references/claude-md-template.md`.

---

## Synthesis Rules

These are the operating principles for how the three docs relate. They're enforced, not suggested.

**Volatility separation.** Never mix intent (PRD, slow) with design (SPEC, per-feature) with agent context (CLAUDE.md, mistake-driven). The fastest-changing content wins and slower content rots inside it.

**One artifact, one volatility.** PRD doesn't include implementation. SPEC doesn't include task lists. CLAUDE.md doesn't include the current sprint's plan.

**Tech-stack lives in SPEC and CLAUDE.md, never in PRD.** This is the most-violated rule; the gate enforces it for PRD. If you find yourself naming a framework in the PRD, you're collapsing the PRD/SPEC split.

**Alternatives or it isn't a SPEC.** "We chose PostgreSQL" is a decision log entry. "We chose PostgreSQL over SQLite because we need concurrent writes; we accept the operational cost" with two real alternatives and a do-nothing baseline is a SPEC. The "why not the alternatives" is what makes the SPEC valuable in 6 months.

**Positive framing over negative.** "Use functional components with hooks" beats "DON'T use class components." Reserve NEVER for genuinely dangerous operations (NEVER delete production data, NEVER commit secrets).

**Mistakes drive CLAUDE.md updates.** If Claude Code makes a mistake the first time and the fix would have lived in CLAUDE.md, add it (one line, positive framing, imperative tone). If Claude Code makes the same mistake a third time despite CLAUDE.md saying not to, the file is too long and primacy has collapsed — extract older content to `agent_docs/`.

**Length is checklist-driven, not page-driven.** A doc is "done" when all required gate sections pass. There's no virtuous length floor; the only hard length ceiling is CLAUDE.md ~200 lines (Anthropic's documented adherence threshold). PRD and SPEC are bounded by completeness, not pages.

---

## Anti-Patterns Quick Reference

Full descriptions and recovery patterns are in `references/anti-patterns.md`. The most common failure modes the gate catches:

| Anti-pattern | Doc | Symptom |
|---|---|---|
| Solution-first writing | PRD | Opens with the design instead of the problem |
| Vague success criteria | PRD / SPEC | "Improve UX" / "Make it faster" with no numbers |
| Tech stack in PRD | PRD | Framework, library, or DB named in PRD |
| No Alternatives Considered | SPEC | One chosen design, no alternatives, no do-nothing baseline |
| Missing cross-cutting concerns | SPEC | No security / privacy / observability sections |
| Auto-generated boilerplate prose | CLAUDE.md | Mirrors README, paragraphs of prose ("Welcome to…", "This is a TypeScript project that…") |
| Instruction overload | CLAUDE.md | >200 lines or >150 discrete instructions |
| File-system maps that go stale | CLAUDE.md | Specific paths beyond top-level directory anchors |
| Secrets in CLAUDE.md | CLAUDE.md | API keys, connection strings, OAuth secrets |
| Suggestion-form instructions | CLAUDE.md | "We generally try to …" instead of imperatives |
| Volatility mixing | All | Task lists in SPEC, current sprint in CLAUDE.md, design in PRD |
| Docs preceding implementation by months | All | Detailed plans / sequences for unstarted work |

---

## Maintenance Cadence

The skill's outputs are not write-once. SPEC and PRD need light upkeep; CLAUDE.md needs mistake-driven additions.

| When | What to do |
|---|---|
| **Mistake-driven (immediate)** | When you re-explain context to Claude Code, capture the explanation as a one-line CLAUDE.md addition before the lesson evaporates. Highest-leverage maintenance move. |
| **Weekly (~30 min)** | Re-read CLAUDE.md. Apply the line-by-line test ("would removing this cause a mistake?") to anything added that week. Sweep for stale paths or commands. |
| **Monthly (~1 hour)** | Re-read PRD and SPEC against actual project state. Has scope crept? Have non-goals held? Has the SPEC's chosen design held up? Update if reality has moved. |

**Three-question triage when a maintenance question comes up mid-session:**
1. Will Claude Code produce wrong output next session because of this? → fix NOW
2. Will I forget this by next week? → write it into next session's seed prompt or handoff.md
3. Is it cosmetic? → defer to weekly review

---

## Brownfield Inventory Subagent

When step 2 of the Core Workflow determines the project is brownfield and no specific file was pointed at, dispatch a general-purpose subagent with this prompt. The summary it returns grounds the interview so you can skip questions the repo already answers.

**Prompt template** (paraphrase or paste; keep it tight):

> Inventory this repo to ground a doc-system interview. Don't read any file deeply unless it's under 50 lines. Produce a structured markdown summary covering:
> - **Stack signals** — language/framework from package manifests (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, etc.) with versions where shown.
> - **Existing docs** — paths and rough line counts of any of: `PRD.md`, `SPEC.md`, `CLAUDE.md`, `README.md`, `AGENTS.md`. Note whether each looks hand-curated or auto-generated.
> - **Top-level directories** — name of each top-level dir with a one-line purpose (no recursion past level 1).
> - **Global conventions** — anything in `~/.claude/CLAUDE.md` that the generated docs should respect (e.g., `docs/milestones.md` workflow, handoff.md ritual).
>
> Keep the summary under 300 words. Don't propose changes; just inventory.

Use the summary to ground the interview ("I see a hand-curated CLAUDE.md and a `pyproject.toml` — confirm you want a new PRD + SPEC and leave CLAUDE.md alone?"). If the inventory surfaces existing PRD/SPEC/CLAUDE.md, treat them as interview context, not as something to regenerate. Never overwrite without confirmation.

If the repo is small enough that an inline scan would clearly fit in context (e.g., under ~10 files at top level, no nested docs), you may skip the subagent and inventory inline — but the default is dispatch.

---

## Reference Files

- `references/prd-template.md` — full PRD template, filled example, per-section rationale, Spec-Kit conventions (FR-NNN, [NEEDS CLARIFICATION], Given/When/Then).
- `references/spec-template.md` — full SPEC template, filled example, per-section rationale, alternatives-considered discipline.
- `references/claude-md-template.md` — full CLAUDE.md template, Anthropic include/exclude lists, length norms, optional artifact-conventions block for solo-dev workflows.
- `references/quality-gate.md` — per-doc pre-flight checklists (enforced before emit) plus cross-doc consistency checks.
- `references/anti-patterns.md` — extended catalog with recovery patterns, organized by doc.

Read these only when needed. The SKILL.md body covers ~90% of common cases; references hold the long-form templates and full discipline notes.
