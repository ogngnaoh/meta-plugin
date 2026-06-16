# Role Recipe — per-role detail + spawn-prompt templates

Spawn every teammate through a **real subagent definition** (the `Agent` tool's agent-type), not a bare prompt. Two things combine on every spawn:

- **The agent definition** sets the role's *standing discipline* and its `tools:` whitelist — so the read-only/edit boundary is enforced by the harness, not merely requested. Use a domain-matching installed agent when one fits, else the shipped generic `meta-plugin:agent-team-{scout,builder,critic,evaluator,reviewer,synthesizer,integrator}`.
- **The 4-part spawn prompt** carries *today's task*: Objective · Output format · Tools & sources · Boundaries. All four are mandatory — omit one and the teammate duplicates work, leaves gaps, or wanders off-task.

Each role below gives its purpose, the definition that backs it, the suggested tool whitelist, model tier, read-only/edit boundary, and a **ready-to-fill spawn-prompt template**. Fill every `<…>` placeholder before spawning. A teammate sees none of your conversation, so the prompt is the *only* channel: put every file path, constraint, and the acceptance criteria into it.

## Resolving a role to an agent definition

Before spawning a role, pick the definition that backs it. **Prefer a domain-matching agent that's already installed** when one genuinely fits; **else fall back to the shipped generic** `meta-plugin:agent-team-{scout,builder,critic,evaluator,reviewer,synthesizer,integrator}`. Installed agents vary per user, so resolve by fit, not a fixed map:

| Role | Prefer an installed agent like… | Generic fallback |
|---|---|---|
| Scout | built-in `Explore` (read-only search) | `meta-plugin:agent-team-scout` |
| Builder | built-in `general-purpose`, or a domain feature-builder | `meta-plugin:agent-team-builder` |
| Reviewer *(default review)* | a small-scope code/skill reviewer | `meta-plugin:agent-team-reviewer` |
| Critic *(high-stakes split)* | a code-review agent (`feature-dev:code-reviewer`, `plugin-dev:skill-reviewer`) | `meta-plugin:agent-team-critic` |
| Evaluator *(high-stakes split)* | a validator (`plugin-dev:plugin-validator` on a plugin) | `meta-plugin:agent-team-evaluator` |
| Synthesizer *(research fan-out)* | *usually no installed match — use the generic* | `meta-plugin:agent-team-synthesizer` |
| Integrator *(parallel slice build)* | built-in `general-purpose` for assembly | `meta-plugin:agent-team-integrator` |

A good installed match brings sharper domain instincts than the generic; the generic guarantees the right tool whitelist and role discipline when nothing fits. Either way, the spawn still carries the full 4-part contract — the definition is the role, the prompt is the task.

---

## Orchestrator *(you — the lead, not a spawned teammate)*

- **Purpose:** decompose the task, route work to teammates, synthesize their returns, hold the two gates.
- **Tools:** coordination tools (`Agent`, `SendMessage`, Task tools, Read/Glob/Grep for sanity checks). **No Edit/Write — but self-imposed:** you're the main session, so nothing strips your tools; holding to orchestrator-only is a *discipline*, not a harness boundary. It matters because the lead quietly becoming the Builder is the most common team failure — route every edit to a Builder.
- **Model:** opus (deep reasoning, routing, synthesis).
- **Mode:** coordinate. You never author the work product.
- **Definition:** none — you are the running session, not a spawned agent. Only the *teammates* below are backed by tool-whitelisted definitions.

---

## Scout *(read-only mapping)*

- **Purpose:** map the problem space and surface risks *before* anything is built. Fan out across independent areas (one Scout per area) when breadth helps.
- **Backed by:** `meta-plugin:agent-team-scout`, or an installed read-only search agent like `Explore`.
- **Tools:** `Read, Glob, Grep, WebSearch, WebFetch`.
- **Model:** sonnet.
- **Mode:** read-only — a Scout never edits.

**Spawn template:**

> **Objective:** Map `<area/subsystem/topic>` to ground a build of `<task>`. Identify how it currently works, the relevant files/contracts, the risks and unknowns, and anything that would change the approach. Do NOT propose a final design — surface what the lead needs to plan.
>
> **Output format:** Write a scout report to `<path, e.g. docs/agent-team/scout-<area>.md>` and return a ≤300-word summary: key findings, risks (ranked), open questions, and the files/paths that matter. Reference the report path; don't paste it back.
>
> **Tools & sources:** Read/Glob/Grep over `<dirs/files>`; WebSearch/WebFetch for `<external docs/specs>` if needed. Start at `<entry points>`.
>
> **Boundaries:** Read-only — make no edits. Don't read files deeply unless they're load-bearing. Stay within `<scope>`; flag anything out of scope rather than chasing it.

---

## Builder *(produces the work product)*

- **Purpose:** produce the work product (code, docs, research, or design) against the plan and acceptance criteria.
- **Backed by:** `meta-plugin:agent-team-builder`, or an installed `general-purpose` / domain feature-builder agent.
- **Tools:** `Read, Glob, Grep, Edit, Write, Bash`.
- **Model:** sonnet → escalate to opus for hard reasoning or architecture.
- **Mode:** edit. **The only role that edits.**
- **Parallel builders:** allowed *only* with strict file ownership or `isolation: worktree`. Never two Builders on the same file.

**Spawn template:**

> **Objective:** Build `<exact deliverable>` to satisfy these acceptance criteria: `<criteria, verbatim>`. Work against the approved plan at `<plan path>`.
>
> **Output format:** Make the change in the working tree and write a short build note to `<path>` listing the files touched and how each criterion is met. Return a ≤200-word summary + the diff/files location. Show any commands you ran to self-check.
>
> **Tools & sources:** Read/Glob/Grep/Edit/Write/Bash. Implement in `<files you own>`; read `<context files>` for patterns/contracts. Run `<build/test/lint cmd>` to self-verify before returning.
>
> **Boundaries:** You own ONLY `<file set>` — do not touch anything else (another Builder owns the rest). Match existing style; minimum change that satisfies the criteria; no scope beyond the task. Don't mark work done without running the self-check.

---

## Reviewer *(default review — finds flaws + renders the verdict)*

The **default review role**: a single fresh-context agent that both finds correctness/requirement defects and renders the pass/fail verdict with evidence. Use it for almost all work — split it into a separate Critic + Evaluator only as a **high-stakes escalation** (next section).

- **Backed by:** `meta-plugin:agent-team-reviewer`, or an installed small-scope code/skill reviewer.
- **Tools:** `Read, Glob, Grep, Bash`. **Model:** sonnet. **Mode:** read-only.

**Spawn template:**

> **Objective:** Review the change at `<diff/output path>` against these acceptance criteria: `<criteria, verbatim>`. Find any correctness/requirement defects AND render a pass/fail verdict.
>
> **Output format:** Findings (severity + location + why), then a per-criterion PASS/FAIL with evidence (real command output). Overall PASS only if every criterion passes. ≤400 words.
>
> **Tools & sources:** Read/Glob/Grep the diff; Bash to run `<checks>` and confirm defects.
>
> **Boundaries:** Read-only. Flag only correctness/requirement issues — no style nitpicks. **A non-blocking (low-severity) flaw still passes** — don't fail a criterion that otherwise meets its bar over a nit; over-reporting stalls the gate. Don't pass a criterion without evidence.

---

## High-stakes split — Critic + Evaluator

Split the single Reviewer into the two distinct roles below **only when the stakes justify an un-anchored second opinion**: high-blast-radius or irreversible changes (security, data migrations, money/auth), compliance/auditability needs, or a finding-set large enough that judging it is its own task. The trigger is **stakes / criteria-type, not task size** — mechanical criteria (tests pass, lint clean) stay with the merged Reviewer. When split, the **Critic** finds (diff + criteria only, no verdict) and the **Evaluator** independently gates with evidence — the finder kept separate from the gate-keeper.

## Critic *(Devil's-Advocate — adversarial finder)*

- **Purpose:** adversarially find flaws, given **only the diff/output + the acceptance criteria** — never the Builder's reasoning, so it judges on the work's own terms.
- **Backed by:** `meta-plugin:agent-team-critic`, or an installed code-review agent (`feature-dev:code-reviewer`, `plugin-dev:skill-reviewer`).
- **Tools:** `Read, Glob, Grep, Bash` (read/run to confirm a flaw; no edits).
- **Model:** sonnet.
- **Mode:** read-only. Fan out by dimension (correctness, security, tests, …) on a wide surface.

**Spawn template:**

> **Objective:** Adversarially review the change at `<diff/output path>` against these acceptance criteria ONLY: `<criteria, verbatim>`. Find correctness and requirement defects — places the work is wrong, incomplete, or violates a criterion.
>
> **Output format:** Return a findings list. Each: severity (high/med/low), file:line or location, what's wrong, why it matters against the criteria. If the work is sound, say so plainly. ≤400 words.
>
> **Tools & sources:** Read/Glob/Grep the diff and the files it touches; Bash to reproduce/confirm a defect. You were NOT given the Builder's reasoning — that is deliberate.
>
> **Boundaries:** Flag ONLY issues affecting correctness or the stated requirements. No style nitpicks, no speculative "could be nicer" — those make the gate un-passable. Read-only: make no edits and propose no rewrites beyond naming the fix.

---

## Evaluator *(gate-keeper — pass/fail with evidence)*

- **Purpose:** score the work against each acceptance criterion, run the checks, and emit **pass/fail + evidence**. Distinct from the Critic: the finder and the gate-keeper are kept separate.
- **Backed by:** `meta-plugin:agent-team-evaluator`, or an installed validator (`plugin-dev:plugin-validator` on a plugin).
- **Tools:** `Read, Glob, Grep, Bash` (must be able to *run* checks).
- **Model:** sonnet.
- **Mode:** read-only.

**Spawn template:**

> **Objective:** Evaluate the change at `<diff/output path>` against each acceptance criterion: `<criteria, verbatim>`. Decide whether each criterion passes, backed by evidence.
>
> **Output format:** A per-criterion table — criterion → PASS/FAIL → evidence (actual command output, test result, or quoted artifact, not an assertion). End with an overall verdict: PASS only if every criterion passes. ≤400 words.
>
> **Tools & sources:** Read the artifact; Bash to run `<test/build/lint/repro cmds>`. Paste the real output as evidence.
>
> **Boundaries:** Read-only — make no edits. Do not fix failures; report them. Do not pass a criterion without showing evidence; "looks correct" is not evidence.

---

## Synthesizer *(merges parallel outputs — research fan-out)*

- **Purpose:** merge the outputs of several parallel Scouts/Researchers (or parallel Builders' notes) into one coherent artifact — a memo, comparison, or landscape. **Surfaces contradictions and gaps across inputs; does not just concatenate.**
- **Backed by:** `meta-plugin:agent-team-synthesizer` (usually no installed match — use the generic).
- **Tools:** `Read, Glob, Grep, Write`.
- **Model:** sonnet → opus when the synthesis requires weighing conflicting evidence into a defensible recommendation.
- **Mode:** read-only over the inputs; **writes only the one synthesis artifact.** Never edits the source reports or any code.

**Spawn template:**

> **Objective:** Merge the parallel research outputs at `<paths, e.g. docs/agent-team/research-*.md>` into one coherent `<deliverable, e.g. decision memo>` that satisfies these acceptance criteria: `<criteria, verbatim — e.g. covers latency-at-scale, self-host cost, ops maturity; yields a defensible recommendation>`. Where the inputs disagree or leave a gap, **say so explicitly** and reconcile or flag it — do not paper over it.
>
> **Output format:** Write the synthesis to `<path>` and return a ≤300-word summary: the recommendation/headline, the key axes of comparison, and any unresolved contradictions or gaps across the inputs. Reference the path; don't paste the whole artifact.
>
> **Tools & sources:** Read/Glob/Grep over the input reports at `<paths>` only. Write only `<synthesis path>`.
>
> **Boundaries:** Read-only except the synthesis artifact — do not edit the source reports or any code. Don't introduce claims not grounded in an input; attribute or mark anything you couldn't corroborate. Don't just concatenate — integrate, and expose the disagreements.

---

## Integrator *(assembles interdependent slices — parallel slice build)*

- **Purpose:** assemble the slices produced by parallel Builders **against the pinned contract**, resolve the integration seams the contract didn't fully specify, and run the integration checks. The single accountable place where parallel pieces become one working whole.
- **Backed by:** `meta-plugin:agent-team-integrator`, or an installed `general-purpose` agent for assembly.
- **Tools:** `Read, Glob, Grep, Edit, Write, Bash`.
- **Model:** sonnet → opus for tricky cross-module reconciliation.
- **Mode:** edit — but only the **integration seams**, not the slices' internals (those belong to their Builders).

**Spawn template:**

> **Objective:** Integrate the parallel slices at `<paths/file sets>` into one working whole against the pinned contract at `<contract path>`. Resolve integration conflicts (mismatched signatures, import paths, overlapping edits at the seams), then run the integration checks and confirm the slices fit.
>
> **Output format:** Make the integrating edits in the working tree, write an integration note to `<path>` (seams resolved, conflicts found, checks run), and return a ≤200-word summary + the location of the integrated diff. **Show the actual output** of the integration checks you ran.
>
> **Tools & sources:** Read/Glob/Grep the slices and the contract; Edit/Write to fix the seams; Bash to run `<build/test/integration cmd>` across all touched call sites. Paste the real output as evidence.
>
> **Boundaries:** Edit only the integration seams — do not rewrite a slice's internals (that's its Builder's responsibility; route real defects back through the lead). Hold to the pinned contract; if a slice violates it, report that rather than silently reshaping the contract. Don't mark integration done without showing the checks pass.
