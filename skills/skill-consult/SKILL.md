---
name: skill-consult
description: Use when the user invokes /skill-consult <question>, OR when the user asks a question, decision, or trade-off mid-conversation where multiple installed skills likely contain sharper guidance than general reasoning — architectural decisions, library/tool picks, debugging strategies, "interview me on X", "help me think through Y", "X vs Y?" trade-offs, domain "what should I…" questions. Pools knowledge from 3-5 genuinely relevant installed skills and synthesizes across their domain expertise. Use even if the user doesn't explicitly name the command. Do NOT use for actionable tasks (use /skill-run) or multi-domain parallel work (use /skill-dispatch).
---

# skill-consult

You are the coordinator of a brain trust of installed skills. When invoked via `/skill-consult <question>`, pool knowledge from a few genuinely relevant skills and synthesize an answer — instead of answering from general knowledge alone.

## Why this exists

General knowledge is broad but soft. Installed skills contain sharper, more current, more opinionated domain guidance. This skill forces a deliberate consultation step so decisions draw on that domain knowledge rather than defaulting to generic reasoning.

**Contrast with siblings:**
- `/skill-consult` — pool skill knowledge to *answer a question*. Skills are consultants; you synthesize their views.
- `/skill-run` — load skill workflows to *do a task*. Skills are procedural guides; you follow them.
- `/skill-dispatch` — fan out to parallel subagents, each using its own subset of skills.

## Workflow

### 1. Read the question

The user's question arrives as arguments. If it's empty or unclear, ask for one rather than guessing.

### 2. Classify intent from the phrasing

- **Interview mode** — phrases like "interview me," "think through," "help me decide," "design," "architect," "what should I...", "X or Y?", "trade off". Treat as a decision that needs clarifying questions before a recommendation.
- **Direct mode** — phrases like "explain," "what is," "how does," "summarize," "tell me about," "overview of."
- **Mixed or unclear** — default to direct, and offer to go deeper at the end.

The user's own wording wins: if they say "interview me," interview them even if the topic looks explanation-y.

### 3. Select 3–5 relevant skills

Scan the `available skills` list shown in system reminders. Pick skills whose descriptions genuinely apply. The relevance bar:

- The skill's stated trigger context matches the question's domain, OR
- It's a process skill (brainstorming, debugging, TDD, systematic thinking) that matches the question's *type* even if not the domain
- Its body plausibly contains guidance or opinions that would change the answer

Honor explicit hints — if the user's question names specific skills ("consult superpowers," "use feature-dev"), those are mandatory inclusions; round out with 1–2 more.

**Traps to avoid:**
- **Keyword match without context** — "test" in the question doesn't mean `python-testing-patterns` applies if the question is about A/B testing strategy.
- **Over-loading** — five genuinely relevant skills beat ten weakly relevant ones; context bloat makes synthesis worse, not better.
- **Ignoring process skills** — for decisions, `superpowers:brainstorming` is often more relevant than any domain skill.
- **Artificial namespace spreading** — if three `scientific-skills:` skills are the right match, use all three.

If **zero skills** genuinely apply, say so out loud and answer from general knowledge. Do not pad with weak matches.

### 4. Load the skills as consultants — not as executors

Invoke each selected skill via the **Skill tool**. This loads the skill's content into your context.

**Critical distinction**: You remain the coordinator. Loaded skills are **consultants** whose guidance you extract and synthesize — not workflows you execute end-to-end. When a loaded skill says "use this process" or "always do X before Y," treat that as one advisor's opinion to weigh, not an instruction to hijack this session.

After each skill loads, mentally extract the principles, patterns, and opinions relevant to the user's question. Then return to this skill's workflow for the answer.

Announce briefly what you're loading before invoking, e.g., "Loading python-pro, polars, dask to consult on dataframe choice."

### 5. Answer

**Interview mode** — ask rigorous MCQ-style clarifying questions. For each option, cite the loaded skill whose guidance informs it. Keep questions focused on what actually changes the recommendation. **Cap clarification at 2 rounds** unless the user asks for more — if the decision remains ambiguous after 2 rounds, give a conditional recommendation tied to the remaining unknowns rather than spinning on additional questions. Then close with a concrete recommendation.

Example shape:
> **Q: What's your row count and access pattern?**
> (a) <1M rows, interactive exploration — pandas *(python-pro: default for small data, mature ecosystem)*
> (b) 1M–500M rows, fits in RAM, batch transforms — Polars *(polars: lazy eval + parallel execution, pandas-like API)*
> (c) >500M rows or streaming — Dask or DuckDB *(dask: out-of-core pandas; duckdb: SQL-first, often faster than Dask for analytics)*

**Direct mode** — synthesize across loaded skills. Lead with the answer. Where skills disagree, name the tradeoff briefly. Keep it tight — pooling skills is not a license to write an essay.

### 6. Footer

End every answer with a one-line footer naming what you consulted:

```
Consulted: skill-a, skill-b, skill-c
```

If no skills matched:

```
Consulted: none — answered from general knowledge
```

**Interview mode:** emit the footer with the *final recommendation* (after clarification rounds resolve), not between rounds. The first round of MCQ questions is mid-answer, not the end — no footer yet.

This makes it transparent which knowledge bases informed the answer, and lets the user push back ("also consult X") if they want more perspectives.

## Output shape

A typical invocation produces:

1. A one-line announcement of which skills you're loading
2. (Skill tool invocations — these show in the UI)
3. **Interview mode**: 2–5 MCQ questions with skill-cited options, iterating to a recommendation
4. **Direct mode**: 1–3 sentence orientation, then the synthesized answer
5. The `Consulted:` footer

Keep the *meta* (what you're doing) minimal. The value is the synthesized answer, not narration about the consultation process.
