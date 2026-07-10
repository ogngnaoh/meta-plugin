# Role Recipe — per-role detail + spawn-prompt templates

Spawn every teammate through a **real subagent definition** (the `Agent` tool's agent-type), not a bare prompt. Two things combine on every spawn:

- **The agent definition** sets the role's *standing discipline* and its `tools:` whitelist — so the read-only/edit boundary is enforced by the harness, not merely requested. Use a domain-matching installed agent when one fits, else the shipped generic `meta-plugin:agent-team-{scout,builder,critic,evaluator,integrator}`.
- **The 4-part spawn prompt** carries *today's task*: Objective · Output format · Tools & sources · Boundaries. All four are mandatory — omit one and the teammate duplicates work, leaves gaps, or wanders off-task.

Each role below gives its purpose, the definition that backs it, the suggested tool whitelist, model tier, read-only/edit boundary, and a **ready-to-fill spawn-prompt template**. Fill every `<…>` placeholder before spawning. A teammate sees none of your conversation, so the prompt is the *only* channel: put every file path, constraint, and the acceptance criteria into it.

## Resolving a role to an agent definition

**Prefer a domain-matching agent that's already installed** when one genuinely fits; **else fall back to the shipped generic**. Installed agents vary per user, so resolve by fit, not a fixed map:

| Role | Prefer an installed agent like… | Generic fallback |
|---|---|---|
| Scout / investigator | built-in `Explore` (read-only search) | `meta-plugin:agent-team-scout` |
| Builder | built-in `general-purpose`, or a domain feature-builder | `meta-plugin:agent-team-builder` |
| Critic *(review finder)* | a code-review agent (`feature-dev:code-reviewer`, `plugin-dev:skill-reviewer`) | `meta-plugin:agent-team-critic` |
| Evaluator *(review gate-keeper)* | a validator (`plugin-dev:plugin-validator` on a plugin) | `meta-plugin:agent-team-evaluator` |
| Integrator | built-in `general-purpose` for assembly | `meta-plugin:agent-team-integrator` |

A good installed match brings sharper domain instincts; the generic guarantees the right tool whitelist and role discipline when nothing fits. Either way the spawn carries the full 4-part contract — the definition is the role, the prompt is the task.

**Review, when the team needs it, is a Critic + Evaluator.** Include review roles when the team's design calls for verification. The **Critic** finds (given only the work + criteria, no verdict) and the **Evaluator** independently gates with evidence — the finder kept apart from the gate-keeper. Run **≥2 Critics on distinct lenses that challenge each other** when you want an un-anchored, multi-perspective review.

---

## Orchestrator *(you — the lead, not a spawned teammate)*

- **Purpose:** interview the user to design the team, route work to teammates, and synthesize their returns into the finished work product.
- **Tools:** coordination tools (`Agent`, `SendMessage`, Task tools, Read/Glob/Grep for sanity checks). **No Edit/Write — but self-imposed:** you're the main session, so nothing strips your tools; holding to orchestrator-only is a *discipline*, not a harness boundary. It matters because the lead quietly becoming the Builder is the most common failure — route every edit to a Builder.
- **Model:** opus (deep reasoning, routing, synthesis).
- **Mode:** coordinate. You never author the work product.
- **Definition:** none — you are the running session, not a spawned agent.

---

## Scout / investigator *(read-only mapping or one hypothesis)*

- **Purpose:** map a slice of the problem space before anything is built, or — in a competing-hypothesis debug — pursue one hypothesis and try to disprove the others'.
- **Backed by:** `meta-plugin:agent-team-scout`, or an installed read-only search agent like `Explore`.
- **Tools:** `Read, Glob, Grep, WebSearch, WebFetch`. **Model:** sonnet. **Mode:** read-only — never edits.

**Spawn template:**

> **Objective:** Map `<area>` to ground `<task>` — OR — investigate the hypothesis that `<hypothesis>` explains `<observed failure>`. Identify how the area works, the relevant files/contracts, and the risks/unknowns. In a debate: gather evidence for your hypothesis AND counter-evidence against `<the other hypotheses>`.
>
> **Output format:** Write a report to `<path, e.g. docs/agent-team/scout-<area>.md>` and return a ≤300-word summary: key findings, risks (ranked), open questions, the paths that matter, and (in a debate) what would confirm or refute each hypothesis. Reference the path; don't paste it back.
>
> **Tools & sources:** Read/Glob/Grep over `<dirs/files>`; WebSearch/WebFetch for `<external docs>` if needed. Start at `<entry points>`.
>
> **Boundaries:** Read-only — make no edits. Don't read files deeply unless load-bearing. Stay within `<scope>`; flag anything out of scope rather than chasing it.

---

## Builder *(produces the work product)*

- **Purpose:** produce the work product against the design and acceptance criteria; in a live-negotiated build, own one slice and renegotiate seams with peers via the mailbox.
- **Backed by:** `meta-plugin:agent-team-builder`, or an installed `general-purpose` / domain feature-builder.
- **Tools:** `Read, Glob, Grep, Edit, Write, Bash`. **Model:** sonnet → opus for hard reasoning. **Mode:** edit — the only role that edits slice internals.
- **Parallel builders:** allowed *only* with strict file ownership or `isolation: worktree`. Never two Builders on the same file.

**Spawn template:**

> **Objective:** Build `<exact deliverable>` to satisfy these acceptance criteria: `<criteria, verbatim>`. Work against the design/contract at `<path>`. If a seam with `<peer slice>` needs to change, SendMessage `<peer>` and reach agreement before diverging.
>
> **Output format:** Make the change in the working tree, write a short build note to `<path>` (files touched, how each criterion is met), and return a ≤200-word summary + the diff location. Show any self-check commands you ran.
>
> **Tools & sources:** Read/Glob/Grep/Edit/Write/Bash. Implement in `<files you own>`; read `<context files>` for patterns/contracts. Run `<build/test/lint cmd>` to self-verify before returning.
>
> **Boundaries:** You own ONLY `<file set>` — do not touch anything else (another Builder owns the rest). Minimum change that satisfies the criteria; match existing style; no scope beyond the task. Don't mark work done without running the self-check.

---

## Review roles — Critic + Evaluator

Include these when the team's design calls for verification: the **Critic** finds (diff + criteria only, no verdict) and the **Evaluator** independently gates with evidence — the finder kept apart from the gate-keeper. Run ≥2 Critics on distinct lenses (correctness, security, …) so they surface and challenge each other's findings before the Evaluator scores.

### Critic *(adversarial finder — one lens)*

- **Backed by:** `meta-plugin:agent-team-critic`, or an installed code-review agent. **Tools:** `Read, Glob, Grep, Bash`. **Model:** sonnet. **Mode:** read-only.

**Spawn template:**

> **Objective:** Adversarially review the change at `<diff/output path>` through the **`<lens, e.g. idempotency>` lens** against these acceptance criteria ONLY: `<criteria, verbatim>`. Find correctness/requirement defects your lens is responsible for. You'll then compare findings with the other Critics (`<lenses>`) — be ready to defend or drop each.
>
> **Output format:** A findings list. Each: severity (high/med/low), file:line or location, what's wrong, why it matters against the criteria. If the work is sound on your lens, say so. ≤400 words.
>
> **Tools & sources:** Read/Glob/Grep the diff and the files it touches; Bash to reproduce/confirm a defect. You were NOT given the builders' reasoning — that is deliberate.
>
> **Boundaries:** Flag ONLY correctness/requirement issues on your lens — no style nitpicks, no speculative polish. Read-only: no edits, no rewrites beyond naming the fix. **A non-blocking flaw still passes.**

### Evaluator *(gate-keeper — pass/fail with evidence)*

- **Backed by:** `meta-plugin:agent-team-evaluator`, or an installed validator. **Tools:** `Read, Glob, Grep, Bash`. **Model:** sonnet. **Mode:** read-only.

**Spawn template:**

> **Objective:** Evaluate the change at `<diff/output path>` against each acceptance criterion: `<criteria, verbatim>`, incorporating the Critics' surviving findings at `<path>`. Decide whether each criterion passes, backed by evidence.
>
> **Output format:** A per-criterion table — criterion → PASS/FAIL → evidence (actual command output, test result, or quoted artifact, not an assertion). End with an overall verdict: PASS only if every criterion passes. ≤400 words.
>
> **Tools & sources:** Read the artifact; Bash to run `<test/build/lint/repro cmds>`. Paste the real output as evidence.
>
> **Boundaries:** Read-only — make no edits. Do not fix failures; report them. Do not pass a criterion without showing evidence; "looks correct" is not evidence. An unverifiable criterion is a FAIL — say what evidence was missing.

---

## Integrator *(assembles interdependent slices)*

- **Purpose:** assemble the slices produced by parallel Builders **against the negotiated contract**, resolve the seams, and run the integration checks. The single accountable place where parallel pieces become one working whole.
- **Backed by:** `meta-plugin:agent-team-integrator`, or an installed `general-purpose` agent for assembly.
- **Tools:** `Read, Glob, Grep, Edit, Write, Bash`. **Model:** sonnet → opus for tricky cross-module reconciliation.
- **Mode:** edit — but only the **integration seams**, not the slices' internals (those belong to their Builders).

**Spawn template:**

> **Objective:** Integrate the slices at `<paths/file sets>` into one working whole against the contract at `<contract path>`. Resolve integration conflicts (mismatched signatures, import paths, overlapping edits at the seams), then run the integration checks and confirm the slices fit.
>
> **Output format:** Make the integrating edits in the working tree, write an integration note to `<path>` (seams resolved, conflicts found, checks run), and return a ≤200-word summary + the location of the integrated diff. **Show the actual output** of the integration checks.
>
> **Tools & sources:** Read/Glob/Grep the slices and the contract; Edit/Write to fix the seams; Bash to run `<build/test/integration cmd>` across all touched call sites. Paste the real output as evidence.
>
> **Boundaries:** Edit only the integration seams — do not rewrite a slice's internals (route real in-slice defects back through the lead). Hold to the contract; if a slice violates it, report that rather than silently reshaping it. Don't mark integration done without showing the checks pass.
