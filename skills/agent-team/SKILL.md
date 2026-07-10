---
name: agent-team
description: Use when the user invokes /agent-team <task>, or asks to build/scaffold/run a role-based agent team for a task. Given the task, you interview the user to design an agent team that fits it — the roles, who owns what, how they coordinate, and explicit acceptance criteria — then scaffold and run that team to produce the work product. Teammates are backed by real subagent definitions that enforce per-role tool whitelists (read-only vs. edit), each spawned with a full 4-part contract; the lead orchestrates and synthesizes but never authors the work product itself. Ships role definitions for Scout, Builder, Critic, Evaluator, and Integrator; the interview picks which the task needs.
---

# agent-team

You are the **team lead** (orchestrator). Given a task, you do two things: **interview** the user to design the team that fits it, then **build** it — scaffold and run the teammates to produce the work product. You decompose, route, and synthesize; **you never author the work product yourself** — that is the Builder's job.

## 1. Interview — design the team that fits

Before spawning anyone, interview the user to establish:

- **The task** — goal, scope, what "done" looks like.
- **The team** — which roles the task needs, who owns what, and how they coordinate. Draw on the common team patterns in `references/archetypes-and-coordination.md` for vocabulary, but design for *this* task: the fewest roles that do the job. If the case wants the work checked, design in review roles (a Critic + Evaluator, or a multi-lens review team); if it doesn't, don't.
- **The acceptance criteria** — explicit, testable conditions that define done. If the team includes verification, these are what it checks against.

Once the design converges, proceed to build. No separate approval step — the interview is the design.

## 2. Build — scaffold and run the team

Spawn each teammate through a **real subagent definition** so its read-only/edit boundary is harness-enforced (a Critic with `tools: Read, Glob, Grep, Bash` literally cannot edit), and pass the full [4-part contract](#the-4-part-spawn-contract). While the team works:

- **Orchestrator-only.** You route and synthesize; every edit goes to a Builder. Nothing strips *your* tools, so this is a discipline you hold — the lead quietly becoming the Builder is the most common failure.
- **Artifacts to disk.** Each teammate writes its output to a file; you pass the *path* to the next, never paste large content agent-to-agent.
- **Strict file ownership.** Every Builder's prompt names the files it owns and forbids the rest. Never two Builders on one file (overwrites). If ownership can't be partitioned, give each `isolation: worktree`.
- **Loop cap.** Any rework loop is capped (default 2–3 rounds); on hitting the cap, stop and surface it rather than grinding.

Synthesize the teammates' returns into the finished work product and report back.

## Roles

One role, one responsibility. The interview picks which the task needs; resolve each to the best-fit definition — a domain-matching installed agent when one fits, else the shipped `meta-plugin:agent-team-{role}`.

| Role | Purpose | Tools | Mode |
|---|---|---|---|
| **Orchestrator** *(you)* | interview, decompose, route, synthesize | no Edit/Write *(self-imposed)* | coordinate |
| **Scout / investigator** | map the space, or pursue one line of inquiry | Read/Glob/Grep/WebSearch/WebFetch | read-only |
| **Builder** | produce the work product | Read/Glob/Grep/Edit/Write/Bash | edit |
| **Critic** | review finder: adversarially flag correctness/requirement defects, given only the work + criteria | Read/Glob/Grep/Bash | read-only |
| **Evaluator** | review gate-keeper: pass/fail per criterion with evidence | Read/Glob/Grep/Bash | read-only |
| **Integrator** | assemble interdependent slices, run integration checks | Read/Glob/Grep/Edit/Write/Bash | edit *(seams only)* |

**Critic + Evaluator are the review roles** — include them when the team's design calls for verification (an adversarial multi-lens review, or checking a build's output). Run ≥2 Critics on distinct lenses when you want them to challenge each other; keep the finder (Critic) separate from the gate-keeper (Evaluator). The **Orchestrator is you** — the main session — so it isn't spawned and can't be tool-stripped; orchestrator-only is a discipline you hold. Per-role spawn templates: `references/role-recipe.md`.

## The 4-part spawn contract

A teammate sees none of your conversation — the prompt is the *only* channel. Every spawn carries all four (miss one and teammates duplicate work, leave gaps, or wander off-task):

1. **Objective** — goal, scope, exact deliverable.
2. **Output format** — what to return and roughly how long.
3. **Tools & sources** — the tool whitelist, what to read, where to look.
4. **Boundaries** — what NOT to do; read-only vs. edit; which files this teammate owns.

Put every file path, constraint, and the acceptance criteria into it.

## Coordination

Default is **hub-and-spoke**: teammates report to you and you relay between them, tracking cross-slice order on your own task list with `blockedBy`. When teammates genuinely must talk to *each other* live — negotiating a shared interface as they build, or challenging each other's findings — escalate to the **peer mailbox** (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, experimental) so they `SendMessage` directly. Details + the parallel-slice coordination playbook: `references/archetypes-and-coordination.md`.

## Traps to avoid

- **Lead does the work itself.** The most common failure — nothing strips your tools, so route every edit to a Builder.
- **Review over-reports.** When the team includes review roles, flag correctness/requirement issues with severity — but a low-severity nit doesn't fail a criterion that otherwise meets its bar.
- **Context telephone.** Don't relay large outputs agent-to-agent; write to disk, pass paths.
- **File overwrites.** Never two Builders on one file — strict ownership or `isolation: worktree`.
- **Looping past the cap.** On cap, stop and surface it — errors compound and tokens blow up.

## Reference files

- `references/archetypes-and-coordination.md` — common team patterns you can design toward (adversarial multi-lens review · competing-hypothesis debugging · live-negotiated parallel build · live steering), with a worked example each, plus the coordination mechanics (hub-and-spoke, `blockedBy`, file ownership, artifacts-to-disk, the Integrator seam, the peer mailbox).
- `references/role-recipe.md` — per-role detail and a ready-to-fill 4-part spawn-prompt template for each of Scout, Builder, Critic, Evaluator, and Integrator.

Read these only when needed; the body above covers the common path.
