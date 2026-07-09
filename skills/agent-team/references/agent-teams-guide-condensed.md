# Agent Teams — Condensed Guide

> **Routing judgment superseded 2026-07-09** by `~/Documents/personal-vault/meta-workflows/2026-07-09-harness-routing-guide.md` — the *decision to use a team at all* is made there, before this skill fires. This digest remains the live operating reference for *running* a team once that decision is made.

A condensed, self-contained digest of the full reference — enough to run a team without leaving this file. When a decision turns on a detail not captured here, consult the source.

**Source:** *A Developer's Guide to Claude Code Agent Teams* — source-verified against official Anthropic documentation and engineering writeups (the subagents, agent-teams, dynamic-workflows, best-practices, and cost references). *(Local copy, if present: `/Users/hoangngo/Documents/personal-vault/meta-workflows/2026-06-16-claude-code-agent-teams-guide.md` — path may change if the vault moves; cite the title.)*

This skill is the *applied, opinionated* version of that guide; the guide is the *reference* behind it. Read the guide to understand the mechanics; use this skill to run a gated team.

## The one principle

**Context is a finite, depleting resource, and multi-agent architecture exists primarily to conserve it.** Add agents only when a task is high-value, breadth-first, parallelizable, or larger than one context window. Otherwise a single agent is simpler, cheaper, and more reliable. *(Guide §TL;DR, §7.)*

## The three tiers (pick the smallest that works)

| Tier | What it is | Best for | Status |
|---|---|---|---|
| **Subagents** | Isolated one-shot delegations; fresh context, return one summary. | Code search, review, focused analysis — protecting main context. | Stable, default. |
| **Agent teams** | Long-lived peer sessions sharing a task list + mailbox. | Full-stack features, parallel investigations, large migrations where subtasks must talk. | Experimental (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, v2.1.32+). |
| **Dynamic workflows** | JS that orchestrates many subagents in the background. | Codebase-wide audits, large batch jobs (10+ agents). | v2.1.154+, paid. |

The defining line: **subagents only report back to the lead and never talk to each other; agent-team teammates share a task list and message directly.** If your subtasks don't need to talk, you want subagents, not a team. *(Guide §1, §3.5–3.6.)*

## Patterns this skill is built from

- **Orchestrator–worker** *(§3.1)* — lead decomposes, delegates to workers, synthesizes. The backbone of this skill.
- **Evaluator–optimizer / fresh-context critic** *(§3.3)* — a separate, fresh-context agent reviews work it didn't write, so it isn't biased toward it. Two cautions, both baked into this skill: don't let the critic run wild (flag only correctness/requirement issues), and have agents *show evidence* rather than assert success. This is the skill's Review phase — a merged Reviewer by default, split into a separate Critic + Evaluator on stakes.
- **Parallel fan-out** *(§3.2)* — independent reads/searches at once (Scouts, Critics by dimension). Works only when paths don't conflict.
- **Sequential pipeline** *(§3.4)* — fixed dependent steps, optional gate between them. The Build → Review spine (split into a Critic + Evaluator on stakes).

## Sizing & cost

- **Start with 3–5 teammates**, give each enough work; "three focused teammates often outperform five scattered ones." *(§3.5.)*
- **Token economics:** chat ≈ 1×, agents ≈ ~4×, multi-agent ≈ ~15×; **Claude Code agent teams ≈ ~7× a standard session in plan mode**, scaling with team size. The headline cost and the reason for the gates. *(§6.)*
- **Scale effort to complexity and say so** — early systems "spawned 50 subagents for simple queries." Right-size the model per role; model choice was a top-three quality driver. *(§5.)*

## Best practices folded into this skill

- **Single responsibility per agent** — a `code-reviewer` and a `security-auditor` beat one `do-everything` agent. *(§5.)*
- **Invest in the delegation prompt** — the 4-part contract (objective, output format, tools/sources, boundaries). Anthropic's cautionary tale: a vague "research the semiconductor shortage" sent subagents chasing the wrong decade and duplicating each other. *(§5.)*
- **Give every agent a way to verify its work; show evidence** — test output, a passing command — not assertions. *(§5.)*
- **Limit tools deliberately; write intermediate artifacts to the filesystem and pass references** to fight context rot and the agent-to-agent "telephone." *(§5, §6.)*

## Experimental limits to respect *(§6)*

Agent teams are experimental — expect rough edges: `/resume` and `/rewind` don't restore in-process teammates; task status can lag (a teammate may forget to mark a task done, blocking dependents); only one team at a time, no nested teams; permissions are fixed at spawn; and **the lead sometimes starts doing the work itself instead of delegating** — the exact failure this skill's orchestrator-only rule guards against. Don't put a team in a critical, unattended pipeline.

> Subagent frontmatter has **no `prompt:` field** — the system prompt is the Markdown body after the closing `---`. The `skills`, `memory`, `initialPrompt`, `color` fields and the "25 concurrent subagents" limit were unconfirmed in official docs at the guide's writing; treat as unverified. *(Guide §2.2–2.3, §3.2.)*
