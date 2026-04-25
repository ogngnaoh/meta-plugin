---
name: skill-run
description: Use when the user invokes /skill-run <task>, OR when the user asks for a task to be actually *done* (not just answered) and specialized skill workflows should shape the execution — implementation work, multi-step planning, refactors, code reviews, or any procedural task where a skill's prescribed workflow beats general defaults. Identifies 2-5 installed skills whose workflows should actively shape execution, loads them, and applies them in-line. Use even if the user doesn't explicitly name the command. Do NOT use for pure questions or explanations (use /skill-consult) or multi-domain parallel work (use /skill-dispatch).
---

# skill-run

You are an executor that works under the active guidance of installed skills. When invoked via `/skill-run <task>`, identify which skills should *shape the execution* of the task, load them via the Skill tool, and then actually apply their workflows, patterns, and guidance while doing the task.

## Why this exists

Claude normally auto-loads skills when it detects a match, but under-triggering happens — especially for composite tasks where several skills should each contribute. This command forces a deliberate "identify the workflow skills that apply, then follow them" step so execution is shaped by specialized knowledge rather than general defaults.

**Contrast with siblings:**
- `/skill-consult` — pool skill knowledge to *answer a question*. Skills are consultants; you synthesize their views.
- `/skill-run` — load skill workflows to *do a task*. Skills are procedural guides; you follow them.
- `/skill-dispatch` — fan out to parallel subagents, each using their own subset of skills.

## Workflow

### 1. Read the task and classify mode

The task arrives as arguments. Classify into one of two modes:

- **Execute mode** (default for tasks): proceed through the workflow below.
- **Redirect mode**: if the input is actually a question, explanation request, or topic inquiry (not an actionable task), suggest `/skill-consult` instead and stop. No footer on redirect.

### 2. Select 2–5 skills that should shape execution

Scan the `available skills` list shown in system reminders. The bar here is stricter than for consult — pick skills that actually change *how* you work, not just what you know:

- **Process / workflow skills come first.** If the task is "plan a feature," `superpowers:writing-plans` and `feature-dev:feature-dev` have real procedures to follow, not just trivia to cite.
- **Domain skills second.** `python-testing-patterns`, `backend-security-coder`, `data-engineering:airflow-dag-patterns`, etc. contribute patterns you'll actively apply while implementing.
- **Prefer workflow skills over reference-only skills.** If a skill's body is mostly static reference material, it's usually better *consulted* than *run*. `/skill-run` is for skills that prescribe process.

Honor explicit user mentions as mandatory inclusions. If the task genuinely matches no skill's workflow, say so and proceed with general best practices — do not fake a match.

### 3. Load the skills and actively follow them

Invoke each selected skill via the **Skill tool**. Unlike `/skill-consult`, here you *do* follow each loaded skill's workflow for the parts of the task it applies to.

**Conflict resolution:**
- If one skill's workflow naturally subsumes another (e.g., `feature-dev:feature-dev` is a superset of `superpowers:writing-plans`), let the broader one dominate and treat the narrower skill's contributions as enrichment for the relevant phases.
- If skills prescribe overlapping steps, merge them — don't run two half-hearted brainstorms, run one informed by both.
- If skills genuinely contradict, name the conflict explicitly and make a reasoned choice.

If a loaded skill has a hard rule ("always do X before Y"), respect it. If it prescribes a format, use it.

**Honor user checkpoints.** If a loaded skill prescribes a pause for user input, confirmation, or sign-off (e.g., `writing-plans` requiring plan approval before implementation, `brainstorming` requiring user validation mid-exploration, TDD-style "run the failing test now" check-ins), honor it — do not skip the checkpoint to keep momentum. The checkpoint exists specifically so the user can course-correct; skipping it defeats the skill's purpose.

### 4. Execute the task

Do the work. As you hit major steps, briefly note which loaded skill is guiding that step — e.g., "Following writing-plans step 3: listing files to create/modify." This keeps the influence of the skills visible to the user without narrating every small move.

### 5. Footer

End with:

```
Applied: skill-a, skill-b, skill-c
```

Or if no skills matched:

```
Applied: none — executed with general best practices
```

**List only skills you actually loaded via the Skill tool during this run.** If you drew on a skill's guidance from prior knowledge without loading it this turn, don't footer-list it — footer honesty is what makes the transparency meaningful.

**Redirect mode:** if Step 1 redirected to `/skill-consult`, emit no footer. The `Applied:` line applies only on the execute path.

## Traps to avoid

- **Keyword-matching a skill that has no applicable workflow.** If the skill is reference-only, it belongs in consult, not run.
- **Over-loading.** Following five workflows is already hard; ten is incoherent. Soft cap at 5.
- **Mechanically running every phase of every skill.** Skills prescribe processes assuming they're the main driver. When several are loaded, you're the conductor — pull the phases that apply, skip the ones that don't.
- **Fighting the user's explicit direction.** If the user names specific skills in the task, those are mandatory; round out with 1–2 more if needed.

## Output shape

1. One-line announcement of which skills you're loading and why
2. (Skill tool invocations — visible in the UI)
3. The actual task execution, with brief in-line notes about which skill is guiding each major step
4. The `Applied:` footer
