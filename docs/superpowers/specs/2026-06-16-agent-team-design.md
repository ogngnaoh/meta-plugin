# agent-team — Design Spec

**Date:** 2026-06-16
**Status:** Approved — adaptive-topology reframe adopted (round 3). Built via a dogfooded team loop.
**Home:** `skills/agent-team/SKILL.md` (+ `references/`, `evals/`) and plugin-level `agents/` in meta-plugin → bump to v2.5.0.
**Grounding:** `/Users/hoangngo/Documents/personal-vault/meta-workflows/2026-06-16-claude-code-agent-teams-guide.md`

## Goal

A manually-invoked Claude Code skill that, given an arbitrary task, **scaffolds and orchestrates an *adaptive* role-based agent team whose topology is designed to fit the task** — turning ad-hoc multi-agent orchestration into a structured, consistent, repeatable workflow the user drives as lead. The team's **shape** (research fan-out · gated build/refine · interdependent parallel slices · staged pipeline · custom) and **coordination tier** are chosen per task in Phase 0, while a fixed set of invariants holds every run. Invoked as `/agent-team <task>`.

## Non-goals

- Not a *bare* one-shot delegation or ungated fan-out — even the research archetype is scoped, synthesized, and human-gated; trivial single-agent jobs stay single-agent.
- Not autonomous/unattended — it stops at human gates and never ships without approval.
- Not code-only — the work product may be docs, research, design, or refactors.
- Does not orchestrate *installed skills*; it orchestrates *agent teammates*.
- Does not *default* to the experimental teammate-to-teammate mailbox — lead-mediated hub-and-spoke is the default; peer messaging is a documented escalation for live-coordination archetypes.

## Adaptive topology — the core principle

`agent-team` is not one fixed loop; it **designs the team topology to fit the task** in Phase 0, then runs it under fixed invariants. The gated build/refine loop is the *default*, not the only shape.

**Team archetypes** — an open, composable set of starting templates (the lead may compose a custom topology):

| Archetype | Shape | For |
|---|---|---|
| **Gated build/refine** *(default)* | Scout → Build → Review, looped, 2 gates | producing a work product to a quality bar |
| **Research fan-out** | parallel Scouts/Researchers → Synthesizer → light verify | breadth-first investigation / comparisons |
| **Parallel slice build** | lead pins the contract/interfaces → N Builders own interdependent slices (dependencies tracked) → integrate → critique/evaluate | complex multi-component features whose slices touch |
| **Pipeline** | brainstorm → plan → build → verify, sequential with gates between stages | staged, dependent work |

**Coordination tier** — first-class and configurable; pick the smallest that works:

- **Lead-mediated / contract-first (default, tier-1):** the lead pins shared interfaces/API up front; teammates relay through the lead; dependencies tracked on the shared task list (`blockedBy`); never two Builders on one file.
- **Peer-to-peer (tier-2, experimental):** when slices must coordinate *live* (e.g., negotiating an API while building), escalate to the experimental agent-teams mailbox so teammates message each other directly.
- **Dynamic workflow (tier-3):** 10+ agents / codebase-wide; background JS orchestration.

## Core behavior — phases & gates

The gated build/refine archetype is the canonical shape; other archetypes reshape the middle phases (1–3) but reuse Phase 0, both gates, and the invariants.

0. **Scope / Interview + topology design** *(skill ↔ user)* — establish the task, explicit **acceptance criteria**, and **design the team topology**: archetype, roles + counts, model tiers, coordination tier, loop cap. Ask clarifying questions; never spawn a teammate before topology + criteria are set.
1. **Scout / Research** *(optional, default on)* — read-only mapping or parallel research of the problem space + risks; may fan out. → **GATE A: user approves the plan/approach/topology** (fires whether or not Scout ran).
2. **Build** — Builder(s) produce the work product. Sequential by default; parallel/interdependent Builders only with strict file ownership or worktree isolation, a pinned contract, and dependency tracking (never same-file).
3. **Review** — *default:* a fresh-context **Reviewer** gets only the diff/output + acceptance criteria, finds correctness/requirement flaws (with severity), then renders **pass/fail + evidence** — a *non-blocking* flaw still passes (this neutralizes the over-report/stall risk). *High-stakes escalation:* split into a separate **Critic** (finds flaws, diff+criteria only) + **Evaluator** (independent criteria-bound gate, shows evidence) when the change is high-blast-radius/irreversible (security, migrations, money/auth), compliance-bound, or the finding-set is large enough to be its own task — a **stakes** trigger, not size. Either way, fan out by dimension if the surface is wide.

↻ For iterative archetypes, repeat the relevant middle phases under a **hard loop cap (default 2–3)**; on cap, stop and escalate. → **GATE B: user approves ship.**

**Termination:** work meets its acceptance criteria *with evidence* AND no open **blocking** (high-severity) correctness/requirement findings AND the user approves at GATE B. Never ships autonomously.

## Default role recipe (configurable; composed into the chosen archetype)

| Role | Purpose | Tools | Model | Mode |
|---|---|---|---|---|
| **Orchestrator** *(lead)* | decompose, route, synthesize, design topology, hold gates | no Edit/Write (self-imposed) | opus | coordinate |
| **Scout / Researcher** | map/research the problem space + risks | Read/Glob/Grep/WebSearch/WebFetch | sonnet | read-only |
| **Builder** | produce the work product | Read/Glob/Grep/Edit/Write/Bash | sonnet→opus | edit |
| **Critic** *(Devil's-Advocate)* | adversarially find flaws — only diff + criteria | Read/Glob/Grep/Bash | sonnet | read-only |
| **Evaluator** | score vs. criteria, run checks, pass/fail + evidence | Read/Glob/Grep/Bash | sonnet | read-only |

Archetype-specific roles compose in as needed — e.g. **Synthesizer** (merges parallel research/build outputs into one coherent artifact; read-only + Write the synthesis doc) for research fan-out, **Integrator** (assembles interdependent slices; edit) for parallel slice build. Single responsibility per role throughout.

## Agent definitions & runtime resolution

Teammate tool boundaries are enforced by spawning each role through a **real subagent definition** carrying a `tools:` whitelist — not by prose. For each role the skill **resolves to the best-fit available definition**: a domain-matching **installed** agent when one fits (`Explore` → Scout; a code-review agent → Critic; `plugin-dev:plugin-validator` → Evaluator on a plugin; `general-purpose` → generic Builder), else the **shipped** `meta-plugin:agent-team-{scout,builder,critic,evaluator,reviewer,synthesizer,integrator}`. The **lead's** no-edit is *self-imposed orchestrator discipline* — the lead is the main session and cannot be tool-stripped; the skill says so honestly.

## Orchestration mechanism

- **Default: lead-mediated hub-and-spoke** — the lead spawns teammates via the `Agent` tool with self-contained prompts and relays between them across rounds via `SendMessage`.
- **Artifacts to disk:** intermediate outputs (scout report, plan/contract, diff, eval report) are written to files; lightweight references are passed between agents to avoid context "telephone."
- **Shared task list (tier 1):** the lead maintains its own task list (Task tools) for visibility, sequencing, and dependency tracking (`blockedBy`); teammate self-claiming is a tier-2 agent-teams feature.
- **Coordination escalation:** climb the tier ladder only when the lower tier's constraint actually bites — peer mailbox for live inter-teammate coordination, dynamic workflow for 10+ agents.

## The 4-part spawn contract (every teammate prompt)

1. **Objective** — goal, scope, exact deliverable.
2. **Output format** — what to return and roughly how long.
3. **Tools & sources** — tool whitelist, what to read, where to look.
4. **Boundaries** — what NOT to do; read-only vs. edit; which files this teammate owns.

Missing any one causes drift (Anthropic multi-agent-research finding).

## Fixed vs configurable

**Fixed — invariants that hold across ALL archetypes:** Phase 0 scoping + topology design before any spawn · both human gates (plan A + ship B) · orchestrator-only lead (self-imposed no-edit) · every spawn carries the 4-part contract · teammates backed by real agent definitions with tool whitelists · artifacts-to-disk + file-reference hand-offs · single-responsibility roles · a loop cap for iterative archetypes · the Critic (when present) sees only diff+criteria · the Evaluator (when present) shows evidence.

**Configurable — designed per task in Phase 0:** the **team archetype / topology** · the **coordination tier** · acceptance criteria *(the critical input)* · roles + counts (skip Scout; merged Reviewer by default, split into Critic+Evaluator on stakes; N parallel/interdependent Builders; add Synthesizer/Integrator) · # parallel scouts/critics + dimensions · model tier per role · loop cap · worktree isolation.

## Pitfalls baked into the skill (mitigations)

- **Lead does the work itself** → orchestrator-only; self-imposed no-edit; route work to the Builder.
- **Critic over-reports → gate never passes** → Critic flags only correctness/requirement issues w/ severity; finder (Critic) kept separate from gate-keeper (Evaluator).
- **Context telephone** → artifacts to disk + file refs + 4-part contract; Critic gets diff+criteria only.
- **Over-spawning / wrong topology** → Phase 0 scales the team to complexity AND picks the archetype that fits — don't run a gated build loop for pure research, or a bare fan-out for interdependent build.
- **Same-file overwrites in parallel build** → strict file ownership / worktree isolation + a pinned contract + dependency tracking.
- **Errors compound / token blowup (~7×)** → hard loop cap + `maxTurns`; gates checkpoint spend; GATE A precedes the expensive build.

## Skill artifact structure

- `skills/agent-team/SKILL.md` — frontmatter (`name`, `description` with when-to-use **and** when-NOT) + house-style body: Overview / Why / Workflow (phases + gates) / **Team archetypes & coordination** / Default role recipe / Agent definitions & resolution / 4-part spawn contract / Fixed vs configurable / Traps / Output shape.
- `skills/agent-team/references/` — on-demand: role-recipe + spawn templates · **archetypes & coordination-tier playbook** · orchestration/escalation · condensed guide pointer.
- `skills/agent-team/evals/evals.json` — sample tasks + expectations (incl. non-default archetypes) so behavior is testable.
- `agents/agent-team-{scout,builder,critic,evaluator,reviewer,synthesizer,integrator}.md` — shipped plugin-level role definitions with real `tools:` whitelists + role prompts, spawned as `meta-plugin:agent-team-*`; the default fallback when no better-matching installed agent fits.
- Updates: `plugin.json` → `2.5.0`; `README.md` components table + reference section.

## Acceptance criteria (→ the Evaluator's evals)

1. On invocation the skill runs a Scope/Interview that establishes acceptance criteria + designs the team topology, asking clarifying questions, **before** spawning any teammate.
2. Teammates are backed by **real subagent definitions** (a domain-matching installed agent when one fits, else a shipped `meta-plugin:agent-team-*`) so tool whitelists / read-only boundaries are genuinely enforced — plus model tier + a full 4-part spawn contract each. The lead's no-edit is framed honestly as self-imposed discipline.
3. The lead is orchestrator-only — it decomposes/routes/synthesizes/designs-topology/gates and does not author the work product itself.
4. It runs both human gates (A plan, B ship) with a loop cap + escalation on cap for iterative archetypes; GATE A is not coupled to the optional Scout.
5. **Review defaults to a single fresh-context Reviewer** that finds correctness/requirement flaws (with severity) then renders pass/fail + evidence (a non-blocking flaw still passes). It **splits into a separate Critic (finds) + Evaluator (gates)** as a **stakes**-based escalation — high-blast-radius/irreversible, compliance, or a large finding-set — NOT a size-based one. Whether merged or split, the reviewer(s) see only diff+criteria and show evidence.
6. It is configurable yet ships consistent opinionated defaults.
7. It terminates only on the user's GATE-B approval; never ships autonomously.
8. The `description` triggers on team/orchestration requests and states when NOT to use it; the body matches meta-plugin SKILL.md conventions and cross-references the guide.
9. **Adaptive topology:** Phase 0 designs the team topology to fit the task (archetype, not just roles); the gated build/refine loop is the DEFAULT but not the only supported shape.
10. **Archetypes:** the skill documents at least four archetypes — gated build/refine, research fan-out, parallel slice build, pipeline — as an open/composable set, each reusing the fixed invariants.
11. **Coordination:** the coordination mechanism is configurable across the tier ladder (lead-mediated/contract-first → experimental peer mailbox → dynamic workflow) and explicitly supports interdependent parallel Builders with dependency tracking (contract-first + shared-task-list `blockedBy` / peer messaging).

## How we build it (dogfood loop)

Round 1 (draft) and round 2 (enforcement + critique fixes) shipped. **Round 3** implements the adaptive-topology reframe: archetypes + coordination tiers + Phase 0 topology design + Synthesizer/Integrator roles + the new acceptance criteria. Then re-critique / re-evaluate → GATE B → integrate (plugin.json 2.5.0 + README) + commit on branch `agent-team-skill`.
