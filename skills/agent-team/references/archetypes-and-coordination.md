# Archetypes & Coordination — the per-shape playbook

`agent-team` doesn't run one fixed loop; in Phase 0 it **designs the topology to fit the task.** This file is the playbook behind that choice: for each archetype, its role set, control-flow, where the two gates sit, its typical coordination tier, and a worked example. Then the coordination-tier detail — most importantly, how to run **interdependent parallel Builders** without overwrites or deadlock.

The archetypes are an **open, composable set** — start from the closest one and adapt, or compose a custom shape. Whatever you build reuses the [fixed invariants](../SKILL.md#fixed-vs-configurable): Phase 0 + topology design, **GATE A** (approve plan/approach) before the expensive work, **GATE B** (approve ship) before anything lands, an orchestrator-only lead, the 4-part contract on every spawn, single-responsibility roles, and a loop cap on any iterative stage. The review rules — fresh context, only diff + criteria, evidence shown (whether one Reviewer or a split Critic + Evaluator) — apply **whenever the Review phase is present**.

---

## 1. Gated build/refine *(default)*

- **For:** producing a work product to a quality bar — features, refactors, docs, a design.
- **Roles:** Scout *(optional)* · Builder · **Reviewer** (split into a separate Critic + Evaluator only as a high-stakes escalation).
- **Control-flow:** mostly **sequential** — Scout → plan → Build → Review, looping Build→Review under the cap. The Reviewer (or, when split, the Critic) may fan out by dimension within a round.
- **Gates:** **GATE A** after the plan (whether or not a Scout ran), before the first Build. **GATE B** after the review passes with evidence and zero open blocking findings.
- **Coordination tier:** **tier 1** (lead-mediated). One Builder; nothing to coordinate live.
- **Worked example — "add a DLQ handler to our Kafka consumer":** Scout maps the consumer loop + retry path → plan → GATE A → Builder writes the handler + wiring + tests → a fresh-context Reviewer gets only the diff + criteria ("poison message lands in DLQ after 3 retries; consumer keeps draining"), finds any defects, runs the integration tests, and renders pass/fail with the pasted output → round 2 if needed → GATE B. *(Tests-pass is a mechanical criterion, so the merged Reviewer suffices — no split.)*

This is the canonical shape; the other three reshape its middle.

---

## 2. Research fan-out

- **For:** breadth-first investigation or comparison where there's **no single artifact to "build"** — the deliverable is knowledge: a memo, a recommendation, a landscape.
- **Roles:** N parallel Scouts/Researchers (one per option/area) · **Synthesizer** · a light verify pass (a Reviewer checking the synthesis against the criteria — *not* a build loop).
- **Control-flow:** **parallel** Scouts → barrier → **Synthesizer merges** → light verify. Fan-in, not a loop.
- **Gates:** **GATE A** approves the *research plan* — the questions, the comparison axes, the criteria for a "defensible" answer — before fanning out (that's where the spend is). **GATE B** approves the synthesized deliverable.
- **Coordination tier:** **tier 1.** Scouts are independent and read-only; they never touch each other's work, so no live coordination is needed.
- **Worked example — "compare pgvector, Qdrant, Pinecone for our RAG workload":** GATE A pins the axes (latency at 10M vectors, self-host cost, ops maturity, a defensible recommendation) → three Scouts run in parallel, one per database, each writing `docs/agent-team/research-<db>.md` → the Synthesizer reads all three, builds one comparison memo, and **surfaces where the sources disagree or leave gaps** rather than concatenating → a verify pass checks every axis is covered and claims are sourced → GATE B.
- **Why not the gated loop:** there's no diff to review and nothing to "pass/fail-build." Forcing a Build→Review loop here wastes effort on work that has nothing to ship; the Synthesizer + a light verify is the right shape.

---

## 3. Parallel slice build *(interdependent components)*

- **For:** complex multi-component features whose slices **touch each other** — shared types, an API both sides call, a schema several modules import.
- **Roles:** lead pins the contract · N **Builders**, each owning one slice · **Integrator** · a **Reviewer** (split into a separate Critic + Evaluator when the change is high-stakes — a migration usually is; see the worked example).
- **Control-flow:** **contract-first, then parallel, then integrate.** The lead pins shared interfaces *before* any Builder starts; Builders run in parallel under **strict file ownership** (or `isolation: worktree`); the Integrator assembles the slices, resolves seams, and runs integration checks; then review over the integrated whole.
- **Gates:** **GATE A** approves the **contract + the slice/ownership map + the dependency order** — this is the highest-leverage gate in the whole skill, because a wrong contract makes every parallel slice wrong at once. **GATE B** approves the integrated, evaluated result.
- **Coordination tier:** **tier 1 by default** (contract-first + `blockedBy` on the shared task list); **escalate to tier 2** only if slices must renegotiate the contract *while building* (see below).
- **Worked example — "extract the billing module out of the monolith (≈40 call sites)":**
  1. **Pin the contract (lead):** define the new package's public surface — exported functions, types, import path — and write it to `docs/agent-team/contract.md`. Nothing builds against a moving target.
  2. **Map ownership + dependencies → GATE A:** Slice 1 = move code into the package (owns `billing/**`); Slice 2 = split the tests (owns `tests/billing/**`); Slice 3 = update the ≈40 call-site imports (owns the call-site files). Slice 3 `blockedBy` Slice 1 (the import path must exist first). User approves contract + map.
  3. **Build in parallel under strict ownership:** Builders 1 and 2 run concurrently (disjoint files); Builder 3 starts when Slice 1 unblocks it. Use `isolation: worktree` if the file sets can't be cleanly partitioned. **Never two Builders on one file** — that's the classic overwrite failure.
  4. **Integrate:** the Integrator pulls the slices together, fixes seams the contract didn't fully specify, and runs the build + full test suite across all call sites, **showing the output**.
  5. **Review (split into Critic + Evaluator — a billing migration is high-blast-radius/irreversible, so the un-anchored second opinion is worth its cost) → GATE B:** the Critic gets the integrated diff + criteria; the Evaluator independently confirms every call site compiles and tests pass with evidence.

---

## 4. Pipeline

- **For:** staged, dependent work where each stage **consumes the last** and a human should be able to stop between stages — e.g. brainstorm → design spec → plan → build → verify.
- **Roles:** one focused role per stage (e.g. a Scout/brainstormer, a planner-Builder, a build-Builder, a Reviewer) — single responsibility per stage.
- **Control-flow:** **strictly sequential** with an optional gate between stages; no fan-out, because each stage needs the previous stage's output.
- **Gates:** the two mandatory gates map onto stage boundaries — **GATE A** at the plan/spec boundary (approve direction before build), **GATE B** before ship. Additional inter-stage checkpoints are optional but cheap.
- **Coordination tier:** **tier 1.** Sequential by nature — there's nothing concurrent to coordinate.
- **Worked example — "design and ship a new onboarding flow":** brainstorm the intent → write a design spec → **GATE A** → plan the implementation → build → verify with evidence → **GATE B**. If a later stage invalidates an earlier one, loop back to that stage under the cap rather than patching forward.

---

## Coordination tiers — pick the smallest that works

Coordination is **configurable and first-class.** Default to the lowest tier; climb only when its constraint actually bites. (The escalation ladder with costs and version gates is in `orchestration-and-escalation.md`.)

### Tier 1 — lead-mediated / contract-first *(default)*

The lead is the hub; teammates are spokes that **don't talk to each other.** This carries almost everything, *including interdependent parallel slices*, via four moves:

- **Contract-first.** Before any parallel Builder starts, the lead pins the shared interfaces/API/schema and writes them to disk (e.g. `docs/agent-team/contract.md`). Each Builder's spawn prompt quotes the contract verbatim. This is what lets independent contexts produce pieces that fit.
- **Dependency tracking via the shared task list.** The lead keeps its own task list (Task tools) and records cross-slice dependencies with **`blockedBy`** — a Builder whose slice depends on another's output isn't spawned until its blocker is done. This is the lead's single-owner list; teammate self-claiming is a tier-2 feature.
- **Strict file ownership.** Every Builder's spawn prompt names the exact file set it owns and forbids touching anything else. Two Builders on one file overwrite each other — the classic team failure. If ownership can't be cleanly partitioned, give each Builder `isolation: worktree`.
- **Integrate through one role.** Parallel slices are assembled by a single **Integrator**, not by each Builder reaching into the others — that keeps the seams in one accountable place.

### Tier 2 — peer-to-peer mailbox *(experimental)*

Escalate **only when slices must coordinate *live*** — the contract genuinely can't be fully pinned up front and Builders need to negotiate it *as they build* (e.g. two sides discovering the API shape together). Enable the experimental agent-teams mailbox (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) so teammates `SendMessage` each other directly. Cost is high (documented ~7× a standard session when teammates run in plan mode, scaling with team size) and it's experimental — so the bar is "tier-1 contract-first actually failed," not "it might be convenient." Most interdependent builds do **not** need this: a well-pinned contract removes the need for live chatter.

### Tier 3 — dynamic workflow

For **10+ agents or codebase-wide** fan-out (audits, large batch migrations) where manual spawning is unwieldy: a background JS orchestration script fans out/in over many subagents and returns one consolidated result, preserved as a reusable artifact. Reach for it when tier-2 manual coordination stops scaling.

---

## Quick chooser

- Deliverable is **knowledge / a comparison**, no artifact to build → **research fan-out** (Scouts → Synthesizer).
- One coherent artifact to a quality bar → **gated build/refine** (the default).
- Several components that **share interfaces and touch** → **parallel slice build** (contract-first → parallel Builders → Integrator).
- **Staged** work where each step feeds the next → **pipeline**.
- None fit cleanly → **compose** a custom topology from the roles, keeping every fixed invariant.

When in doubt, prefer the simplest shape and the lowest coordination tier; you can always escalate, and the gates make over-investment recoverable.
