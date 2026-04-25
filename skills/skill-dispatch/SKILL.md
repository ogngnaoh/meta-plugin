---
name: skill-dispatch
description: Use when the user invokes /skill-dispatch <task>, OR when the user asks for multi-domain / multi-lens / parallel work where several independent analyses need to run concurrently — multi-angle audits (security + perf + tests), multi-domain planning (backend + frontend + data), comparative analyses (library A vs B vs C, each researched independently), or any task the user describes as parallel. Decomposes into 2-6 independent subtasks, selects which skills each subagent should invoke, and delegates parallel dispatch to superpowers:dispatching-parallel-agents with those per-subagent skill instructions baked in. Use even if the user doesn't explicitly name the command. Do NOT use for single-domain work (use /skill-run), sequentially dependent steps, or pure questions (use /skill-consult).
---

# skill-dispatch

You are a dispatch *planner*. When invoked via `/skill-dispatch <task>`, decompose the work into independent subtasks, decide which skills each subtask's worker should invoke based on its purpose and context, then hand execution off to `superpowers:dispatching-parallel-agents` — that skill owns the actual parallel-dispatch mechanics. Your contribution is the *skill-per-subagent mapping*. When all subagents return, synthesize.

## Why this exists

Some tasks span multiple domains or lenses — a code audit needs security, performance, and test-coverage views simultaneously; a system design needs backend, frontend, and data perspectives. Running these serially is slow and causes cross-contamination of reasoning. Dispatching to parallel subagents, each equipped with the right skills for its narrow job, produces sharper per-domain analysis that the main agent then synthesizes.

**Contrast with siblings:**
- `/skill-consult` — pool skill knowledge in the main context to answer a question
- `/skill-run` — load skill workflows in the main context to execute a task
- `/skill-dispatch` — fan out to parallel subagents, each using its own subset of skills

**How this composes with `superpowers:dispatching-parallel-agents`:** that skill knows *how* to spawn Agent subagents in parallel and coordinate their returns. It does not know *which skills* each subagent should load. `skill-dispatch` fills that gap: it handles task decomposition and skill-per-subagent selection, then invokes the superpowers skill as the execution engine.

## Workflow

### 1. Read the task and classify mode

The task arrives as arguments. Classify into one of two modes before doing any other work:

- **Dispatch mode** — the task has 2+ independent lenses/domains and parallelism makes sense. Proceed through the workflow below.
- **Redirect mode** — stop after suggesting the appropriate sibling or declining: single-domain → suggest `/skill-run`; sequentially dependent → explain the dependency and decline; pure question → suggest `/skill-consult`. No footer on redirect.

Sanity-check fit against these categories:

**Good dispatch candidates:**
- Multi-lens audits (security + perf + tests + docs)
- Multi-domain planning (backend API + frontend UI + data pipeline)
- Parallel research / investigation across areas
- Comparative analyses (library A vs B vs C, each researched independently)
- Anything the user explicitly describes as "in parallel"

**Bad candidates** — suggest an alternative and stop:
- Implementing a single feature → `/skill-run`
- Debugging one specific bug → `/skill-run`
- A pure question → `/skill-consult`
- Subtasks that depend on each other's output (must be sequential)

### 2. Decompose into 2–6 subtasks

Each subtask must be:
- **Independent** — doesn't consume another subtask's output
- **Purposeful** — clear goal and a concrete deliverable
- **Bounded** — fits cleanly under 1–3 skills' territory

Too few (1) means nothing to parallelize. Too many (>6) usually means over-decomposition; if you're tempted to make every skill its own subagent, reconsider — that's not the point.

### 3. Pick skills per subagent

For each subtask, pick 1–4 skills from the `available skills` list whose descriptions match that subtask's *purpose* and *context*. Subagents can share skills or not — each runs independently.

The selection is tighter than in `/skill-consult` or `/skill-run` because a subagent has a narrow job and limited context budget. Pick only skills that directly serve that subagent's purpose.

### 4. Delegate execution to superpowers:dispatching-parallel-agents

Invoke `superpowers:dispatching-parallel-agents` via the Skill tool. That skill owns the actual parallel-dispatch mechanics — spawning Agent subagents, running them concurrently, handling returns. Do not re-implement dispatch yourself; let superpowers drive.

Your contribution to that dispatch is the **skill-per-subagent mapping** you computed in step 3. When constructing each subagent's prompt (following superpowers' workflow), bake in explicit skill-invocation instructions so the subagent loads the right skills before working. Include in every subagent prompt:

1. **Subtask statement** — goal, scope, expected deliverable
2. **Skill instructions** — list the skills to invoke via the Skill tool, one sentence each explaining what it's for in this subtask. Subagents don't auto-pick skills as aggressively as the main agent; be explicit.
3. **Context transfer** — file paths, constraints, prior findings, the broader goal. Subagents don't see the main conversation, so the prompt must be self-contained.
4. **Output format** — what to return and rough length, so synthesis is tractable.

**Example subagent prompt shape:**

> Subtask: audit `src/api/` for security issues.
>
> Before analyzing, invoke these skills via the Skill tool:
> - `backend-security-coder` — apply its input-validation, auth, and secret-handling patterns
> - `python-error-handling` — use its silent-failure patterns to flag swallowed exceptions
>
> Context: FastAPI service with JWT auth. Routes in `src/api/routes/`, auth helpers in `src/api/auth.py`. Ignore test files. Focus on endpoints that accept user input or issue DB queries.
>
> Output: punch list of findings with severity (high/med/low), file:line reference, brief description, recommended fix. Under 400 words.

If superpowers' workflow gives options on `subagent_type`, prefer specialized subagents (`feature-dev:code-reviewer`, `pr-review-toolkit:*`, `data-engineering:*`, etc.) when one matches a subtask's purpose. Fall back to `general-purpose` for open investigations.

**Failure mode — `superpowers:dispatching-parallel-agents` unavailable.** If that skill isn't installed or can't be invoked (Skill tool returns an error, or it's missing from the available-skills list), stop and tell the user — do NOT fall back to calling the `Agent` tool directly from `skill-dispatch`. The skill-dispatch / superpowers separation is what keeps the dispatch traceable and the subagent-prompt discipline consistent; bypassing it silently turns this skill into a black box. If the user explicitly asks to proceed without superpowers, that's a separate, opt-in decision.

### 5. Synthesize

When all subagents return, synthesize — don't just concatenate their reports. Look for:

- **Cross-cutting issues** — a concern flagged by multiple subagents
- **Contradictions** — one says X is fine, another flags it (surface the conflict, don't hide it)
- **Gaps** — a subagent that returned nothing meaningful (worth noting, may mean the decomposition was wrong)
- **Priorities** — across all findings, what matters most

**Failure mode — partial subagent failures.** If one or more subagents return errors or empty output, synthesize from what *did* return, explicitly call out which subtasks failed and why (timeout / crash / no findings / missing files), and append a `(failed)` marker next to the failed subtask in the `Dispatched:` footer. Do NOT silently drop failed subtasks — the user needs to know the audit is incomplete so they can decide whether to re-run.

### 6. Footer

End with a dispatch summary:

```
Dispatched:
- <subtask-1>: skill-a, skill-b
- <subtask-2>: skill-c
- <subtask-3>: skill-d, skill-e
```

**Redirect mode:** if Step 1 rejected the task (single-domain → `/skill-run`, sequential dependencies, or a pure question), emit no footer. The `Dispatched:` line applies only when a dispatch actually happened.

## Traps to avoid

- **Forcing parallelism on sequential work.** If subtasks depend on each other, this is the wrong skill. Say so and stop.
- **Re-implementing dispatch.** Don't call the Agent tool directly from `skill-dispatch`; let `superpowers:dispatching-parallel-agents` drive the actual spawning. Your job is decomposition + skill-per-subagent selection + synthesis.
- **Over-provisioning subagents.** A subagent whose job is "check for SQL injection" doesn't need 6 skills; 1–2 well-chosen ones beat a broad load.
- **Under-provisioning.** A subagent with zero skills is just default Claude — if a subtask doesn't merit at least one skill, it probably shouldn't be its own subagent.
- **Omitting context.** Subagents can't see the main conversation. File paths, constraints, and the overall goal must be in the prompt or the subagent will flail.
- **Concatenating instead of synthesizing.** The value of dispatch is cross-domain synthesis; just pasting subagent reports wastes that.

## Output shape

1. One-line decomposition summary ("Fanning out to 3 subagents: security, perf, tests")
2. Per-subagent skill mapping (shown briefly, so the user sees what each worker will load)
3. Load `superpowers:dispatching-parallel-agents` and follow its workflow to spawn the subagents concurrently
4. (Subagents return)
5. Synthesized answer — cross-cutting issues, contradictions, priorities
6. The `Dispatched:` footer
