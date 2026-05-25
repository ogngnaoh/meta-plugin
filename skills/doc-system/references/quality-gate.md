# Quality Gate — Pre-Flight Checklist per Document

This is the hard gate the skill applies before emitting any document. **If a check fails, do not emit the doc.** Either resolve the gap by interview or insert a `[NEEDS CLARIFICATION: ...]` marker that names what's missing and why it's deferred.

The gate exists because the most common failure modes for each doc are structural — missing metrics, missing alternatives, hidden secrets — and they're cheap to catch at emit time, expensive to catch in production.

## How the gate is applied

1. Draft the document in full per its template.
2. Walk every check below for the doc type.
3. For each check that fails:
   - If the answer is knowable, interview the user for the missing input.
   - If the answer is genuinely unresolved, write the gap as an inline `[NEEDS CLARIFICATION: <what>]` marker at the point of impact, and let the corresponding section reference it explicitly (e.g., "FR-006 is gated on [NEEDS CLARIFICATION: ...]"). The gate passes if every failure is named via a NEEDS CLARIFICATION marker — silence is the failure mode the gate prevents.
4. Only after every check has either passed or been explicitly deferred, emit the doc.

Surface the gate's verdict to the user before emitting (e.g., "PRD passed gate with 2 [NEEDS CLARIFICATION] markers — proceed?"). The user sees what was deferred, can choose to resolve it now or accept the markers.

---

## PRD.md gate

| # | Check | Why it matters |
|---|---|---|
| P1 | Problem section is non-empty and describes user pain, not the solution. | Solution-first PRDs lock in the wrong build. The problem is the artifact's anchor. |
| P2 | At least 1 target user / persona is named. | A PRD without a named user can't be validated. "Me" is a valid answer for solo work, but must appear. |
| P3 | At least 1 goal has a measurable, time-bounded metric (numeric, not "improve UX"). | Without metrics, you can't tell if the build worked after launch. |
| P4 | At least 3 non-goals are listed. | Non-goals are the highest-leverage scope-creep prevention. Less than 3 means scope wasn't considered. |
| P5 | At least 1 user story exists with Given/When/Then acceptance scenarios. | Stories without acceptance scenarios are wishes, not requirements. |
| P6 | At least 1 functional requirement uses `FR-NNN` numbering and "System MUST …" form. | Numbered, testable FRs are the unit of contract between PRD and SPEC. |
| P7 | **No technology mentions.** No framework names, no library names, no database choices, no API SDKs. | Tech-stack mentions belong in SPEC. Mixing them collapses the PRD/SPEC split. |
| P8 | Every ambiguous claim either resolved or marked `[NEEDS CLARIFICATION: ...]` at the point of impact. | Silent assumptions become production bugs. NEEDS CLARIFICATION makes the gap visible. |

**Fail rule:** any check failing without an explicit NEEDS CLARIFICATION marker blocks emit.

---

## SPEC.md gate

| # | Check | Why it matters |
|---|---|---|
| S1 | TL;DR is present and ≤5 sentences. | Readers stop after the first sentence; the TL;DR carries the load. |
| S2 | At least 1 goal has a numeric value (latency, throughput, scale, time). | "Improve latency" is weak. Without numbers, the design can't be evaluated. |
| S3 | At least 1 non-goal is listed. | Mirrors PRD's discipline; prevents reviewers from debating features that weren't on the table. |
| S4 | Alternatives Considered section exists with **≥2 alternatives** plus an explicit **"do nothing" baseline**. | This is the SPEC's signature section. Without it, the doc is an implementation manual, not a design doc. The do-nothing baseline prevents marginal features that didn't need to ship. |
| S5 | Security, Privacy, and Observability each addressed (even one line). "N/A — <reason>" is acceptable if explicitly stated. | Google's canonical templates require these to be touched. Silence here is how compliance and incident issues get found late. |
| S6 | Rollout / migration plan present (or marked N/A with reason for greenfield single-deploy). | Outages at launch trace to missing rollout plans more often than to missing tests. |
| S7 | Testing & validation section is non-empty (unit + integration coverage at minimum, plus how we'll know in prod). | A SPEC without a test plan is an aspiration. |
| S8 | Related PRD is linked (path or URL). | Decouples SPEC's how from PRD's why; reviewers can cross-check. |

**Fail rule:** any check failing without an explicit NEEDS CLARIFICATION marker blocks emit. S4 is non-deferrable — alternatives must be enumerated; deferring "consider alternatives" is not acceptable.

---

## CLAUDE.md gate

| # | Check | Why it matters |
|---|---|---|
| C1 | No secrets, API keys, connection strings, or anything sensitive. | Leaks via git history. Never. |
| C2 | Build / test / typecheck / lint commands present (each on its own line, project-specific). | The single highest-leverage section; most agent mistakes trace to missing or wrong commands. |
| C3 | No file paths beyond top-level directory anchors (no `src/auth/handlers.ts` — `src/auth/` is fine). | Specific paths rot the moment a file moves. CLAUDE.md describes patterns, not specifics. |
| C4 | No rules a linter, formatter, or `.editorconfig` already enforces. | If CI blocks a violation, the rule belongs in CI, not CLAUDE.md. Duplication causes drift. |
| C5 | No auto-`/init`-style boilerplate prose ("This is a TypeScript project that…", "Welcome to…", etc.). | ETH Zurich measured `/init`-style content *reduces* agent success 2-3% versus hand-written. Delete on sight. |
| C6 | No stale "Current Focus" or "What we're working on now" section. | That's what handoff.md is for; CLAUDE.md should contain only stable context. |
| C7 | Imperative tone throughout — no "we generally try to …" or "it might be a good idea to …". | Suggestion-form instructions are ignored by the model. Only imperatives steer behavior. |
| C8 | Length ≤200 lines (soft warning if 150–200; harder warning if >200, never a hard block). | Anthropic's documented threshold for adherence; HumanLayer's analysis says even shorter is better. |

**Fail rule:** C1 is **always** a hard block (no exceptions — never emit a CLAUDE.md with secrets). C2 is a hard block (commands missing makes the file useless). C3–C7 each block emit unless explicitly justified inline (e.g., "deliberate exception: project lacks a linter for X"). C8 is a warning only.

---

## Cross-doc consistency check (when emitting more than one doc in a single session)

| Check | Why it matters |
|---|---|
| Tech-stack mentions in PRD ↔ SPEC: 0 in PRD, all in SPEC. | The PRD/SPEC split collapses if tech leaks left. |
| Non-goals in PRD do not contradict scope in SPEC. | The SPEC must not silently widen scope past what the PRD allowed. |
| FR-NNN references in SPEC point to actual FR-NNN entries in PRD. | Traceability — every FR in the PRD should be addressable by the SPEC. |
| CLAUDE.md "Pointers" section references the PRD and SPEC that were just emitted. | Without pointers, Claude Code never reads the PRD or SPEC. |

These checks only apply when the user is emitting multiple docs in the same session. Single-doc emits skip them.
