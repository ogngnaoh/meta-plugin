# Orchestration & Escalation

How the lead actually coordinates the team, and when to climb the multi-agent ladder. Default to the **smallest tier that solves the problem** — a single agent is cheapest, fastest, and most reliable; add machinery only when the task is high-value, breadth-first, parallelizable, or larger than one context window.

## Default mechanism: lead-mediated hub-and-spoke

The lead is the hub; teammates are spokes that don't talk to each other directly. This is the default because it keeps coordination simple and the run traceable.

1. **Spawn through a tool-whitelisted definition, with a self-contained prompt.** The lead spawns each teammate via the `Agent` tool, resolving the role to a real subagent definition (a domain-matching installed agent, else the shipped `meta-plugin:agent-team-{role}`) so its read-only/edit boundary is enforced by the harness — then passing a full 4-part-contract prompt for the task. The teammate sees none of the lead's conversation — the prompt is the only channel.
2. **Artifacts to disk, references between agents.** Every intermediate output — scout report, plan, diff/build note, eval report — is written to a file (e.g. under `docs/agent-team/`). The lead passes the *path*, not the pasted content, into the next teammate's prompt. This is the single most important anti-telephone move: large outputs relayed agent-to-agent degrade into a game of telephone, and each fresh context re-bloats if you paste instead of reference.
3. **Relay across rounds.** Between rounds the lead reads each teammate's return, synthesizes, and routes the next prompt — e.g. it feeds the review findings back into the Builder's next spawn. Use `SendMessage` to continue a still-running teammate with its context intact; use a fresh `Agent` call when you want a clean context (always fresh for the reviewer).
4. **The lead's task list (tier 1).** The lead maintains its *own* task list (Task tools) for visibility, sequencing, and **cross-slice dependency tracking** — recording which slice `blockedBy` which, so an interdependent parallel build (the *parallel slice build* archetype) doesn't spawn a Builder before its blocker is done. This is a single-owner list the lead reads and updates. **Teammates self-claiming from a shared, file-locked task list is a tier-2 agent-teams feature** (see the escalation ladder) — don't assume it at the default tier.

Why hub-and-spoke and not a teammate-to-teammate mailbox by default: the mailbox (real-time teammate messaging) is **experimental and off by default**, and most build/review work is relay-shaped, not real-time-conversation-shaped. Reach for the mailbox only when subtasks genuinely must coordinate live (see escalation below).

## The escalation ladder — smallest tier that works

| Tier | Mechanism | Reach for it when | Cost |
|---|---|---|---|
| **1. Subagents** *(default)* | Lead spawns isolated, one-shot `Agent` calls; each returns a single summary. Hub-and-spoke. | The roles relay through the lead and don't need to talk to each other in real time — i.e. almost all agent-team work. | Low — only summaries return. |
| **2. Agent teams** *(experimental)* | Long-lived peer Claude Code sessions sharing a task list + mailbox; enable with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (v2.1.32+). | Subtasks must coordinate *live* — e.g. parallel Builders negotiating a shared API contract as they build, or reviewers comparing findings in real time. | High — ~7× a standard session in plan mode; scales with team size. |
| **3. Dynamic workflows** | Claude writes a JS orchestration script that fans out/in over many subagents in the background, returning one consolidated result (v2.1.154+, paid). | Coordinating **10+ agents** with real logic — codebase-wide audits, large batch migrations — where manual spawning is unwieldy and you want the orchestration preserved as a reusable artifact. | Scales with subagent count. |

**Default to tier 1.** The 4-part contract, artifacts-to-disk, and the gated loop all work identically at tier 1; tiers 2 and 3 are documented escalations, not the default. Climb only when the tier-1 constraint actually bites (no real-time coordination at tier 1; manual spawning gets unwieldy past ~10 agents at tier 2).

**Tier ↔ archetype mapping** (the per-archetype playbook is in `archetypes-and-coordination.md`): *research fan-out*, *pipeline*, and the default *gated build/refine* all sit at **tier 1** — their roles relay through the lead and never coordinate live. *Parallel slice build* is **tier 1 by default too** (contract-first + `blockedBy` dependency tracking + strict file ownership) and escalates to **tier 2** only when slices must renegotiate the contract *while building*. Codebase-wide fan-outs of 10+ agents go to **tier 3**. The smallest tier that satisfies the archetype's coordination need is the right one.

## Cost & checkpoint discipline

- **Token economics are the headline cost.** Multi-agent runs cost roughly an order of magnitude more than a chat (agent teams ~7× a standard session in plan mode). The value of the task has to justify the spend — that's *why* GATE A precedes the expensive build/loop and GATE B precedes ship.
- **Cap the loop.** Errors compound across stateful, multi-step runs; a hard loop cap (default 2–3) plus per-teammate `maxTurns` keeps a bad trajectory from burning the budget. On cap, stop and escalate to the user.
- **Right-size the model per role.** Cheap/deterministic work → haiku; general build/review → sonnet; deep reasoning/architecture/synthesis → opus. Model choice is one of the top drivers of quality and cost.
- **Limit tools deliberately.** Spawned teammates carry a real `tools:` whitelist from their definition: a read-only reviewer with `Read/Glob/Grep` literally cannot edit, can't be tempted off-task, and carries less context. Grant the minimum that does the job; use `isolation: worktree` for any parallel role that edits. (The *lead* is the exception — it's the main session and can't be tool-stripped, so its orchestrator-only no-edit is self-imposed discipline, not an enforced limit.)

## Don't-parallelize checklist

Before fanning out, confirm the work is actually parallelizable. Bad fits — keep them sequential or single-agent:

- Same-file edits (two Builders overwrite each other).
- Tightly-coupled or high-dependency steps where each consumes the last.
- Tasks small enough that one agent finishes before a team finishes spawning.

Most coding tasks have fewer truly parallel subtasks than research does. Fan out for **breadth** (independent reads, independent files); keep **dependent** work in a sequential chain.
