---
name: agent-team
description: Use when the user invokes /agent-team <task>, OR when a task is substantial enough to warrant a persistent, role-based agent team the user drives as lead — multi-file features, large refactors or migrations, deep multi-source research, or design work where a gated refine → review loop beats a one-shot answer. Designs the team's topology to fit the task (research fan-out · gated build/refine · interdependent parallel slices · staged pipeline) and scaffolds single-responsibility teammates (Scout, Builder, Reviewer — split into Critic + Evaluator for high-stakes work — plus Synthesizer/Integrator as needed) backed by real subagent definitions that enforce per-role tool whitelists and read-only/edit boundaries, each spawned with a full 4-part contract; the lead orchestrates and synthesizes but never authors the work product, stops at two human gates (plan, then ship), caps the loop, and never ships autonomously. Use even if the user doesn't name the command. Do NOT use for a single quick delegation, a one-shot parallel fan-out, a pure question, or a trivial single-file edit a lone agent handles faster — a team's ~7× token cost only pays off when the work is high-value, parallelizable, or larger than one context window.
---

# agent-team

You are the **team lead** (orchestrator). When invoked via `/agent-team <task>`, you scaffold a role-based agent team and **design its topology to fit the task** — defaulting to a gated **Scope → Scout → Build → Review** loop, but reshaping into a research fan-out, interdependent parallel slices, or a staged pipeline when the task calls for it. Whatever the shape, the run stops at two human gates, with a hard loop cap on any iterative stage. You decompose, route, and synthesize; **you never author the work product yourself** — that is the Builder's job. The work product may be code, docs, research, or a design.

## Overview

A team turns ad-hoc multi-agent orchestration into a repeatable, gated workflow. The skill **designs the team's topology to fit the task** — the gated **Build → Review** loop is the *default archetype, not the only shape*: pure research wants a fan-out into a synthesis, interdependent components want parallel slice-builders against a pinned contract, staged work wants a pipeline. Phase 0 picks the archetype, the roles and counts, the model tiers, and the coordination tier; a fixed set of invariants (the two gates, the orchestrator-only lead, the 4-part contract, a loop cap on any iterative stage) holds every run regardless of shape. So a one-file rename, a 40-site migration, and a three-database comparison each get the machinery they need and no more.

The defining property: each teammate runs in a **fresh, isolated context** and is steered entirely by its spawn prompt. Each is spawned through a **real subagent definition** carrying a `tools:` whitelist, so a role's read-only/edit boundary is enforced by the harness, not merely requested in prose. Intermediate artifacts (scout report, plan, diff, eval report) are written to disk and passed by lightweight file reference, never pasted agent-to-agent — that is how you avoid the "game of telephone" as outputs hop between contexts.

## Why this exists

Multi-agent orchestration done by hand drifts: the lead starts coding instead of delegating, the critic nitpicks until no gate ever passes, simple tasks get five agents, the wrong shape gets forced onto the task (a gated build loop for pure research, a bare fan-out for interdependent slices), and context degrades as large outputs are relayed between sessions. This skill bakes the mitigations in — an orchestrator-only lead, a fresh-context reviewer whose discipline is *a non-blocking flaw still passes* (split into a separate finder and gate-keeper only when stakes demand it), a topology and team scaled to the task, artifacts-to-disk, and a loop cap with human checkpoints — so the structure is consistent and the failure modes are designed out rather than rediscovered each time.

It is **standalone**: it orchestrates *agent teammates*, not installed skills, and it is for tasks worth a *persistent, gated, iterative* team — not a one-shot fan-out or a single delegation.

## Workflow

The phases below are the **default (gated build/refine) archetype.** Other archetypes reshape the middle phases (1–3) but reuse Phase 0, both gates, and the invariants — see [Team archetypes & coordination](#team-archetypes--coordination). **Never spawn a teammate before Phase 0 has set the topology and explicit acceptance criteria.**

### 0. Scope / Interview + topology design *(you ↔ user — no teammates yet)*

Establish four things by asking clarifying questions:

- **The task** — goal, scope, what "done" looks like.
- **The team topology** — pick the **archetype** that fits (gated build/refine *[default]* · research fan-out · parallel slice build · pipeline · a custom composition), then the roles + counts, model tiers, the **coordination tier**, and the loop cap (see [Team archetypes & coordination](#team-archetypes--coordination) and [Fixed vs configurable](#fixed-vs-configurable)). Once the roles are set, **resolve each to the best-fit available agent definition** before spawning (see [Default role recipe](#default-role-recipe)).
- **The acceptance criteria** — *the critical input.* Explicit, testable conditions the reviewer will score against. If you can't name them, you can't evaluate; do not proceed until they're set.

Scale the *topology* to the work, not just the headcount: don't run a gated build loop for pure research (fan out Scouts → a Synthesizer), or a bare fan-out for interdependent components (pin the contract → parallel slice-builders → integrate). Within the chosen shape, skip Scout when the problem is well-understood, run the default single **Reviewer** unless the stakes call for splitting it into a separate Critic + Evaluator (see [Phase 3](#3-review)), and plan N parallel Builders only for genuinely independent or contract-pinned, separately-owned files. Fewest roles by default.

### 1. Scout *(optional)* → **GATE A**

When the problem isn't already well-understood, spawn one or more read-only Scouts to map it and surface risks before anything is built; fan out across independent areas if breadth helps. They write a scout report to disk, which you synthesize into a plan. When the problem *is* well-understood, skip Scout and write the plan directly.

→ **GATE A — the user approves the plan/approach before any build begins** — fires whether or not a Scout ran. Skipping Scout never skips the gate; it only changes how you arrived at the plan. This checkpoints spend *before* the expensive loop.

### 2. Build

The Builder produces the work product against the plan + acceptance criteria, writing output (diff/files) to disk. **Sequential by default.** Spin up parallel Builders only with strict file ownership or worktree isolation — never two Builders on the same file (overwrites are the classic team failure).

### 3. Review

**Default — one fresh-context Reviewer.** Spawn a single Reviewer given **only the diff/output + the acceptance criteria** — never the Builder's reasoning, so it judges the work on its own terms. It finds correctness/requirement flaws (each with a **severity**), then renders **pass/fail + evidence** (actual command output — tests, build, lint, grep — not an assertion of success). The discipline that keeps the gate passable: **a non-blocking flaw still passes** — a low-severity nit, once noted, does not fail a criterion that otherwise meets its bar. Fan out by dimension (correctness, security, tests, …) if the surface is wide.

**High-stakes escalation — split into Critic + Evaluator.** When the change is high-blast-radius or irreversible (security, data migrations, money/auth), compliance/auditability-bound, or the finding-set is large enough that judging it is its own task, split the review into two **distinct roles**: a **Critic** that finds flaws (given only diff + criteria, severity-tagged, no verdict) and an independent **Evaluator** that gates each criterion pass/fail with evidence. Keeping the finder separate from the gate-keeper buys an un-anchored second opinion — worth its per-cycle cost only when the stakes justify it. **The trigger is stakes / criteria-type, not task size:** mechanical criteria (tests pass, lint clean, zero stale references) → the merged Reviewer is plenty; judgment-laden or high-stakes criteria → split. Either way the review runs in **fresh context, sees only diff + criteria, and shows evidence.**

### ↻ Loop + **GATE B**

Repeat **Build → Review (2 → 3)** under a **hard loop cap (default 2–3 rounds)**, recycling the review findings to the Builder each round. **On hitting the cap, stop and escalate to the user** — do not silently keep looping.

**Terminate only when all three hold:** the review passes every acceptance criterion *with evidence*, AND there are no open **blocking** correctness/requirement findings, AND **the user approves at GATE B (ship).** The team never ships autonomously.

## Team archetypes & coordination

The gated loop above is the **default archetype, not the only shape.** In Phase 0 you pick the archetype that fits the task — these are an **open, composable set**; you may compose a custom topology. Every archetype reuses the [fixed invariants](#fixed-vs-configurable): Phase 0, both gates, the orchestrator-only lead, the 4-part contract, single-responsibility roles, and a loop cap on any iterative stage. The review rules — fresh context, only diff + criteria, evidence shown — apply whenever the Review phase is present. Full playbook — role set, control-flow, gate placement, coordination tier, and a worked example per archetype — in **`references/archetypes-and-coordination.md`**.

| Archetype | Shape | Reach for it when |
|---|---|---|
| **Gated build/refine** *(default)* | Scout → Build → Review, looped under a cap, 2 gates | producing a work product to a quality bar — features, refactors, docs |
| **Research fan-out** | parallel Scouts/Researchers → Synthesizer → light verify | breadth-first investigation or comparisons; no single artifact to "build" |
| **Parallel slice build** | lead pins the contract/interfaces → N Builders own interdependent slices (deps tracked) → Integrator → review | complex multi-component features whose slices touch |
| **Pipeline** | brainstorm → plan → build → verify, sequential with a gate between stages | staged, dependent work where each stage consumes the last |

**Coordination tier — pick the smallest that works.** First-class and configurable; climb the ladder only when the lower tier's constraint actually bites (full ladder + costs in **`references/orchestration-and-escalation.md`**):

- **Tier 1 — lead-mediated / contract-first (default):** the lead pins shared interfaces up front, relays between teammates, and tracks dependencies on its own task list (`blockedBy`); never two Builders on one file. Handles almost all work, *including* interdependent parallel slices.
- **Tier 2 — peer-to-peer mailbox (experimental):** when slices must coordinate *live* (e.g. negotiating an API while both sides build), escalate to the experimental agent-teams mailbox so teammates message each other directly.
- **Tier 3 — dynamic workflow:** 10+ agents or codebase-wide; background JS orchestration that fans out/in and returns one consolidated result.

## Default role recipe

One role, one responsibility. **Review defaults to a single fresh-context Reviewer** that finds the flaws and then renders the verdict; **split it into a separate Critic + Evaluator only as a high-stakes escalation** (see [Phase 3](#3-review) and [Fixed vs configurable](#fixed-vs-configurable)). Ready-to-fill spawn-prompt templates for every role are in **`references/role-recipe.md`**.

| Role | Purpose | Tools | Model | Mode |
|---|---|---|---|---|
| **Orchestrator** *(you, the lead)* | decompose, route, synthesize, hold the gates | no Edit/Write *(self-imposed)* | opus | coordinate |
| **Scout** | map the problem space + risks before committing | Read/Glob/Grep/WebSearch/WebFetch | sonnet | read-only |
| **Builder** | produce the work product | Read/Glob/Grep/Edit/Write/Bash | sonnet→opus | edit |
| **Reviewer** *(default review)* | find correctness/requirement flaws, then render pass/fail + evidence — a non-blocking flaw still passes | Read/Glob/Grep/Bash | sonnet | read-only |

**High-stakes split — Critic + Evaluator.** When the stakes justify an un-anchored second opinion (high-blast-radius/irreversible, compliance, or a finding-set that's its own task), replace the single Reviewer with two distinct roles — the finder kept separate from the gate-keeper:

| Role | Purpose | Tools | Model | Mode |
|---|---|---|---|---|
| **Critic** *(Devil's-Advocate)* | adversarially find flaws — given only diff + criteria | Read/Glob/Grep/Bash | sonnet | read-only |
| **Evaluator** | score vs. criteria, run checks, pass/fail + evidence | Read/Glob/Grep/Bash | sonnet | read-only |

**Archetype-specific roles compose in as needed** (backed by `meta-plugin:agent-team-synthesizer` / `-integrator`): a **Synthesizer** merges parallel research/build outputs into one coherent artifact — surfacing contradictions and gaps across inputs, not just concatenating — and is read-only except the synthesis doc (`Read/Glob/Grep` + `Write`); an **Integrator** assembles interdependent slices against the pinned contract, resolves integration conflicts, and runs integration checks (`Read/Glob/Grep/Edit/Write/Bash`). Single responsibility per role throughout; templates for every role are in **`references/role-recipe.md`**.

**Spawn each role through a real subagent definition — that is what makes the tool boundaries real.** A read-only Reviewer spawned with `tools: Read, Glob, Grep, Bash` literally cannot edit; the whitelist is enforced by the harness, not promised in prose. Two things combine on every spawn:

- **The agent definition** sets the role's *standing discipline + tool whitelist* (e.g. a Reviewer that only ever flags correctness/requirement defects, read-only).
- **The 4-part spawn prompt** carries the *per-task contract* — this objective, this output, these sources, these boundaries (see [The 4-part spawn contract](#the-4-part-spawn-contract)).

**Resolve each role to the best-fit available definition.** Prefer a domain-matching agent that's *already installed* when one fits the role; else fall back to the shipped generic `meta-plugin:agent-team-{role}`. Installed agents vary per user, so match by fit — don't assume a fixed map. Examples:

| Role | Prefer an installed agent like… | Generic fallback |
|---|---|---|
| Scout | built-in `Explore` (read-only search) | `meta-plugin:agent-team-scout` |
| Builder | built-in `general-purpose` | `meta-plugin:agent-team-builder` |
| Reviewer *(default review)* | a small-scope code/skill reviewer | `meta-plugin:agent-team-reviewer` |
| Critic *(high-stakes split)* | a code-review agent (`feature-dev:code-reviewer`, `plugin-dev:skill-reviewer`) | `meta-plugin:agent-team-critic` |
| Evaluator *(high-stakes split)* | a validator (`plugin-dev:plugin-validator` on a plugin) | `meta-plugin:agent-team-evaluator` |

Whichever definition you pick, **still pass the full 4-part contract**: the definition is the role, the prompt is today's task. The **Orchestrator is you** — the main session — so it isn't spawned from a definition and can't be tool-stripped; its no-edit is a discipline you hold (next section), not a harness restriction.

## The 4-part spawn contract

The agent definition gives a role its standing discipline and tool whitelist; **this contract gives it today's task.** Both ride on every spawn — the definition is the role, the prompt is the assignment. Missing any of the four parts causes agents to duplicate work, leave gaps, or wander off-task (Anthropic's multi-agent-research finding).

1. **Objective** — the goal, scope, and the exact deliverable expected.
2. **Output format** — what to return and roughly how long, so synthesis stays tractable.
3. **Tools & sources** — the tool whitelist, what to read, where to look.
4. **Boundaries** — what NOT to do; read-only vs. edit; which files this teammate owns.

Because a teammate sees none of your conversation, the prompt is the *only* channel — put every file path, constraint, and the acceptance criteria into it. Full per-role templates: **`references/role-recipe.md`**.

**You, the lead, are the exception.** You're the main session, so nothing strips your tools — orchestrator-only is therefore a **discipline you hold, not a limit the harness enforces.** Route every edit to a Builder instead of making it yourself; the reason it matters is that the lead quietly becoming the Builder is the single most common team failure. Teammates' boundaries *are* enforced (tool-whitelisted definitions); yours is on you.

## Fixed vs configurable

**Fixed — invariants that hold across ALL archetypes (this is the consistency the skill exists to provide):**
- Phase 0 scoping + topology design before any spawn · both human gates (plan A + ship B) · the orchestrator-only lead (self-imposed no-edit) · every spawn carries the 4-part contract · teammates backed by real agent definitions with tool whitelists · artifacts-to-disk + file-reference hand-offs · single-responsibility roles · a loop cap on any iterative stage. **When the Review phase is present:** the review runs in fresh context, sees only diff + criteria, and shows evidence — whether merged into one Reviewer or split into Critic + Evaluator. Builder/Integrator/Synthesizer write the work product; reviewer roles stay read-only.

**Configurable — designed per task in Phase 0:**
- the **team archetype / topology** (gated build/refine · research fan-out · parallel slice build · pipeline · custom) · the **coordination tier** (lead-mediated/contract-first → experimental peer mailbox → dynamic workflow) · the acceptance/eval criteria *(the critical input)* · which roles + team size (skip Scout when well-understood; **merged Reviewer by default, split into a separate Critic + Evaluator on stakes**; N parallel/interdependent Builders; add a Synthesizer or Integrator) · number of parallel scouts/critics and their dimensions · model tier per role · the loop cap · worktree isolation.

The default orchestration mechanism (lead-mediated hub-and-spoke), the coordination-tier ladder, and when to climb from subagents to agent-teams to a dynamic workflow live in **`references/orchestration-and-escalation.md`**; the per-archetype playbook lives in **`references/archetypes-and-coordination.md`**.

## Traps to avoid

- **Lead does the work itself.** The single most common team failure. You *can* edit — you're the main session, nothing strips your tools — which is exactly why orchestrator-only is a discipline you have to hold, not a guardrail you can lean on. When tempted to "just fix it," route it to the Builder. (Teammates' boundaries *are* enforced — they're spawned through tool-whitelisted definitions; yours is on you.)
- **Review over-reports → the gate never passes.** A reviewer told to find gaps will always find some. The default Reviewer's discipline is **a non-blocking flaw still passes** — flag correctness/requirement issues with severity, but a low-severity nit doesn't fail a criterion that otherwise meets its bar. Splitting into a separate Critic + Evaluator (finder kept apart from gate-keeper) is the *high-stakes escalation* for when an un-anchored second opinion is worth its per-cycle cost — not the default.
- **Context telephone.** Don't relay large outputs agent-to-agent. Write artifacts to disk; pass file references. The reviewer gets diff + criteria only.
- **Over-spawning on simple tasks.** Five scattered teammates lose to three focused ones. Let the interview scale the team down — sometimes the honest answer is "this is a single-agent job."
- **Forcing the wrong topology.** A gated build loop run over pure research wastes a Reviewer on work that has nothing to ship; a bare fan-out over interdependent slices produces pieces that don't fit together. Phase 0 picks the *archetype* before the roles — research fans out to a Synthesizer; interdependent components pin a contract, build in parallel under strict file ownership, then integrate.
- **Skipping a gate to move faster.** GATE A protects you from building the wrong thing; GATE B is the only exit. Neither is optional.
- **Looping past the cap.** Errors compound and tokens blow up (~7× a normal session). On cap, stop and escalate — don't keep grinding.

## Output shape

This is the default-archetype shape; other archetypes adapt the middle steps (e.g. research fan-out reports per-Scout findings → a Synthesizer's merged artifact instead of a Build/Review round), but the Scope recap, both gates, and the loop-cap behavior on any iterative stage are constant.

1. **Scope recap** — the task, the chosen **topology** (archetype + roles + counts + models + coordination tier + loop cap), and the explicit acceptance criteria, stated back for confirmation before any spawn.
2. **Plan** — the approach and surfaced risks (from the Scout report if one ran, written directly if not) → **pause at GATE A.**
3. **Per round** — a terse status line per teammate (Builder produced X; Reviewer flagged N issues by severity and rendered PASS/FAIL with evidence — or, when split for high stakes, Critic flagged N then Evaluator gated PASS/FAIL), plus what's recycling into the next round.
4. **Ship proposal** — the review's pass-with-evidence (Reviewer, or Evaluator when split), zero open blocking findings, the round count vs. cap → **pause at GATE B.**
5. **On cap without pass** — stop, summarize what's still failing, and escalate the decision to the user.

## Reference files

- `references/archetypes-and-coordination.md` — the per-archetype playbook: role set, control-flow, where the gates sit, the typical coordination tier, and a worked example for each of the four archetypes — including the interdependent parallel-build case (contract-first, dependency tracking, file ownership, tier-2 escalation).
- `references/role-recipe.md` — per-role detail (purpose, suggested tools, model tier, read-only/edit) and a ready-to-fill 4-part spawn-prompt template for each of Scout, Builder, the default Reviewer, the high-stakes Critic + Evaluator split, and the archetype roles Synthesizer and Integrator.
- `references/orchestration-and-escalation.md` — the default hub-and-spoke mechanism, the lead's tier-1 task list, artifacts-to-disk discipline, and the "smallest tier that works" coordination-tier ladder (subagents → agent-teams → dynamic workflows).
- `references/agent-teams-guide-condensed.md` — a condensed pointer to the full agent-teams guide: the three tiers, sizing, token economics, and the documented experimental limits.

Read these only when needed; the SKILL.md body covers the common path.
