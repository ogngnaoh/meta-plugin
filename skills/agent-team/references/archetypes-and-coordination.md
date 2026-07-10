# Team patterns & coordination — a design menu

The interview designs the team that fits your task; this file is the menu it draws on. Each pattern below is a common team shape — its role set, control-flow, and a worked example — followed by the coordination mechanics that carry any of them. Mix and adapt: these are examples to design toward, not a fixed catalog.

Every pattern reuses the discipline the SKILL body sets: an orchestrator-only lead, the 4-part contract on every spawn, single-responsibility roles, artifacts-to-disk hand-offs, strict file ownership, and a loop cap. When the work is reviewed, the review runs in fresh context, sees only the output + acceptance criteria, and shows evidence.

---

## 1. Adversarial multi-lens review

- **For:** high-stakes work where one reviewer isn't enough — you want independent lenses that **surface and challenge each other's findings**.
- **Roles:** N **Critics** (one per lens: correctness · security · perf · data-integrity · …) · **Evaluator**.
- **Control-flow:** Critics review in parallel, then **compare/challenge** each other's findings (peer mailbox) so a weak or duplicated finding gets knocked down and a missed dimension gets caught; the Evaluator then renders per-criterion pass/fail with evidence.
- **Coordination:** peer mailbox — the challenge *between* Critics is the point.
- **Worked example — "review the payments refactor before we ship":** pin four lenses (correctness, money-rounding, idempotency, auth) + criteria → four Critics review the diff in parallel, each handed only diff + criteria, then reconcile: the idempotency Critic's "double-charge on retry" survives cross-challenge, two overlapping style flags get dropped → the Evaluator confirms each criterion with real test/repro output.

---

## 2. Competing-hypothesis debugging

- **For:** a stubborn bug where the failure mode is genuinely unknown and you want rival explanations pressure-tested against each other rather than one line of inquiry.
- **Roles:** N **investigators** (Scout-backed, one per hypothesis) · **Evaluator**.
- **Control-flow:** each investigator pursues one hypothesis and, via the mailbox, **tries to disprove the others'** ("scientific debate"); survivors are the ones no one could refute. The Evaluator (or you) confirms the surviving hypothesis with a reproduction as evidence before any fix is built.
- **Coordination:** peer mailbox — disproving each other *is* the coordination.
- **Worked example — "intermittent 500s under load, cause unknown":** frame three hypotheses (connection-pool exhaustion · a race in the cache-fill · an upstream timeout) → three investigators dig in parallel and message counter-evidence at each other; pool-exhaustion is refuted by metrics one surfaces, the race survives every challenge → Evaluator reproduces the race deterministically.

---

## 3. Live-negotiated parallel build

- **For:** interdependent components whose shared interface is **discovered as they build** — the sides can't fully pin the contract up front.
- **Roles:** lead seeds the contract · N **Builders**, each owning one slice, negotiating seams live · **Integrator** · a review.
- **Control-flow:** seed the contract → parallel Builders under **strict file ownership** (or `isolation: worktree`) who **`SendMessage` each other to renegotiate seams** as they surface → Integrator assembles and runs integration checks → review over the integrated whole.
- **Coordination:** peer mailbox for the live seam negotiation. *If the contract can be pinned up front, the Builders don't need to talk — pin it and run them hub-and-spoke instead (simpler and cheaper).*
- **Worked example — "add a streaming protocol two services co-design":** seed a draft frame format → map ownership (Service A owns `producer/**`, B owns `consumer/**`) → both build in parallel; when B discovers the frame needs a sequence field, it messages A and they amend the contract live → Integrator wires them and runs an end-to-end round-trip with real output → review.

---

## 4. Live human steering

- **For:** several independent workers you want to **watch and redirect mid-run** across panes — the value is your live judgment in the loop.
- **Roles:** any of the above, spawned as long-lived teammates you steer.
- **Control-flow:** teammates work in parallel; you observe, correct, re-scope, and reprioritise between turns rather than waiting for a batch to finish.
- **Coordination:** panes + mailbox.

---

## Coordination mechanics

**Default relay is hub-and-spoke; escalate to the mailbox only where a pattern above needs it.** You are the hub: you spawn each teammate through a tool-whitelisted definition with a self-contained 4-part prompt, read each return, synthesize, and route the next. Teammates message each other directly (peer mailbox) only for the challenge/negotiation a pattern requires; everything else relays through you.

- **Artifacts to disk, references between agents.** Every intermediate output — scout report, diff, findings, integration note — is written to a file (e.g. under `docs/agent-team/`); you pass the *path*, not the pasted content. This is the single most important anti-telephone move.
- **The lead's task list (`blockedBy`).** You keep your own task list for sequencing and cross-slice dependencies — a Builder whose slice depends on another's isn't spawned until its blocker is done.
- **Strict file ownership.** Every Builder's prompt names the exact files it owns and forbids the rest. Two Builders on one file overwrite each other — the classic failure. If ownership can't be partitioned, give each `isolation: worktree`.
- **Integrate through one role.** Parallel slices are assembled by a single **Integrator**, not by each Builder reaching into the others — the seams stay in one accountable place.

## Coordination tiers

| Tier | Mechanism | Reach for it when | Cost |
|---|---|---|---|
| **Hub-and-spoke** *(default)* | Teammates report to you; you relay and track `blockedBy`. | Almost everything — teammates don't need to talk to each other. | Lowest. |
| **Peer mailbox** | Long-lived teammates share a task list + mailbox; `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (experimental). | Teammates must challenge/negotiate live, or you steer panes. | ~7× a standard session *in plan mode*; significantly more tokens otherwise (not quantified by docs). |
| **Dynamic workflow** | Background JS orchestrates many subagents, returns one consolidated result. | 10+ agents / codebase-wide, where manual spawning is unwieldy. | Scales with subagent count. |

## Don't-parallelize checklist

Before fanning out, confirm the work is actually parallel. Bad fits — keep sequential or single-agent: same-file edits (overwrites); tightly-coupled steps where each consumes the last; tasks small enough that one agent finishes before a team finishes spawning. Most coding tasks have fewer truly parallel subtasks than research does.

## Quick chooser

- Independent lenses that **challenge each other** on one artifact → **adversarial multi-lens review**.
- Unknown failure mode, want **rival explanations pressure-tested** → **competing-hypothesis debugging**.
- Interdependent components whose **interface is discovered live** → **live-negotiated parallel build** (pin the contract instead and it's a simple hub-and-spoke build).
- Several workers you'll **watch and redirect live** → **live human steering**.
