# Anti-Patterns Catalog

Extended catalog of documentation anti-patterns. The SKILL.md body has the abridged table; this file has full descriptions with examples and recovery patterns, organized by document.

When auditing existing docs, use this as a checklist. When generating new docs, use it as a list of things NOT to do — the quality gate (`quality-gate.md`) checks for several of these directly.

## Table of Contents
1. [PRD anti-patterns](#prd-antipatterns)
2. [SPEC anti-patterns](#spec-antipatterns)
3. [CLAUDE.md anti-patterns](#claudemd-antipatterns)
4. [System-level anti-patterns (cross-doc)](#system-level-antipatterns)

---

<a name="prd-antipatterns"></a>
## PRD Anti-Patterns

### PRD-1. Solution-first writing
**Symptom:** PRD opens with the proposed design, the chosen framework, or a sketch of the UI — before naming the problem.

**Why it matters:** Engineering creativity is eliminated; the team locks in the wrong build. Marty Cagan calls this "the single most common discovery-stage failure."

**Fix:** Restart with the problem paragraph. The first thing a reader sees should be "Who has this pain, when, and why current solutions fail." Solution / approach comes much later (and most of it lives in SPEC.md, not PRD.md).

### PRD-2. Vague success criteria
**Symptom:** Goals read "improve UX," "make it faster," "be more reliable." No numbers, no time bounds.

**Why it matters:** After launch, no one can agree whether it worked. The PRD becomes unfalsifiable. Sondra Orozco names missing measurable metrics as the most common rookie PRD mistake.

**Fix:** Rewrite each goal with a concrete numeric target and a time bound. "Improve performance" becomes "Reduce P95 page-load on profile view from 2.1s to <1s within 30 days of launch."

### PRD-3. No non-goals section (or fewer than 3)
**Symptom:** PRD has no explicit out-of-scope list, or lists only "polish" / "we'll see."

**Why it matters:** Scope creep enters before sprint 1. Every elite PRD template (Kevin Yien's Square template, Lenny's 1-Pager, Intercom's Intermission, Shape Up) carries an explicit non-goals section because their absence is the leading cause of post-kickoff scope creep.

**Fix:** Write at least 3 non-goals. Force yourself: what is this PRD *not* trying to do that a reasonable reader might assume it does?

### PRD-4. Tech-stack mentions in the PRD
**Symptom:** PRD names a framework, library, database, or specific API.

**Why it matters:** The PRD's job is *what the user experiences*; the SPEC's job is *how the system delivers it*. Mixing them collapses the split and rots the PRD as the tech changes.

**Fix:** Move every tech-stack mention to SPEC.md. The PRD should be readable in 3 years even if the entire stack has been replaced.

### PRD-5. Burying open questions
**Symptom:** Open questions hidden in an appendix at the bottom, while the body of the PRD reads as if everything's resolved.

**Why it matters:** Silent assumptions become production bugs. Spec Kit's `[NEEDS CLARIFICATION: ...]` pattern surfaces ambiguity at the point of impact — inside the FR or story that depends on it — rather than letting it become an invisible assumption.

**Fix:** When drafting a requirement that depends on an unresolved decision, write the marker inline: "FR-006: System MUST authenticate users via [NEEDS CLARIFICATION: auth method not specified — email/password, SSO, OAuth?]." Cross-cutting unresolved items can live in the Open Questions section; local ones stay inline.

### PRD-6. Length bloat (the 50-page Word doc)
**Symptom:** PRD runs 30+ pages, every section padded with hedging prose, every FR followed by half a page of explanation.

**Why it matters:** "Few will read and is impossible to test" (Cagan). Length bloat hides muddled reasoning behind volume.

**Fix:** Apply the line-by-line test to each section: would removing this line change a build decision? If no, cut.

---

<a name="spec-antipatterns"></a>
## SPEC Anti-Patterns

### SPEC-1. Implementation manual masquerading as design doc
**Symptom:** SPEC is full of code snippets, function signatures, exhaustive APIs — but no trade-offs, no alternatives, no design rationale.

**Why it matters:** "Could have been a PR. Design issues are found post-merge, by which point they're expensive." (Industrial Empathy)

**Fix:** Cut the implementation detail and add what's missing: the Alternatives Considered section, the trade-offs that led to the chosen design, the cross-cutting concerns. If the implementation detail belongs anywhere, it's in the code itself with comments — not in the SPEC.

### SPEC-2. No Alternatives Considered section
**Symptom:** SPEC describes one chosen design. No alt A, no alt B, no do-nothing baseline.

**Why it matters:** A reviewer can't judge whether the chosen design is best. The doc fails its primary job: "demonstrate that the chosen design is the best of a real set" (Industrial Empathy).

**Fix:** Add ≥2 serious alternatives plus the do-nothing baseline. For each: what it is, why rejected (with the actual trade-off, not a strawman).

### SPEC-3. Vague goals
**Symptom:** Goals read "improve performance," "make it scalable," "be reliable."

**Why it matters:** Project can't be evaluated; nobody knows when it's done. Mirrors PRD-2.

**Fix:** Rewrite each goal as numeric and testable. "Reduce P95 from 2.1s to <1s at 1000 req/s."

### SPEC-4. Missing non-goals
**Symptom:** SPEC has no non-goals section, or lists only "things obviously not in this design."

**Why it matters:** Reviewers debate features that weren't on the table. Daisuke Shimamoto's framing: "Non-goals aren't negated goals like 'the system shouldn't crash' — they're things that could reasonably be goals, but are explicitly chosen not to be."

**Fix:** Write non-goals that a thoughtful reviewer might otherwise assume are in scope.

### SPEC-5. Missing rollout / migration plan
**Symptom:** No phased rollout, no feature flags, no kill switch, no data migration plan.

**Why it matters:** Outage at launch; rollback impossible. The "two-phase deploy" pattern (Google) is standard practice for zero-downtime cutovers and is missing from most first-draft SPECs.

**Fix:** Add the phased rollout section. For greenfield single-deploy work, write "N/A — single deploy, no flag" so the reviewer sees you considered it.

### SPEC-6. No cross-cutting review
**Symptom:** SPEC has no Security section, no Privacy section, no Observability section.

**Why it matters:** Compliance / incident issues get found late, after the design has been frozen. Google's canonical templates require these to be touched even if briefly — "such parts of design documents are reviewed by experts in those domains" (Software Engineering at Google, Ch. 10).

**Fix:** Add a Cross-cutting concerns section with at least one line per topic. "N/A — <reason>" is acceptable when truly N/A; silence is not.

### SPEC-7. Skipping the "do nothing" baseline
**Symptom:** Alternatives Considered has alt A and alt B, but no do-nothing baseline.

**Why it matters:** Team builds something marginal that didn't need to be built. The do-nothing baseline forces you to articulate: what's actually bad about the status quo? Sometimes the answer is "not much," and that's the SPEC's job to surface.

**Fix:** Add the do-nothing baseline. What happens if we ship nothing? Why is that worse? If "why worse" is hard to fill in, that's the signal.

---

<a name="claudemd-antipatterns"></a>
## CLAUDE.md Anti-Patterns

### CLAUDE-1. Auto-generated boilerplate prose committed un-edited
**Symptom:** CLAUDE.md mirrors README structure, paragraphs of prose, sections like "Welcome" or "Getting Started," restates project name three times.

**Why it matters:** ETH Zurich (2026) tested 138 tasks across 12 Python repos and found that LLM-generated agent context files *reduced* agent success rates by 2-3% compared to hand-written ones. Both forms increased token cost by 20%+. Auto-generation feels productive but is net-negative.

**Fix:** Delete the auto-generated content. Hand-write from the template in `claude-md-template.md`. Apply the line-by-line test for every line.

### CLAUDE-2. Instruction overload
**Symptom:** CLAUDE.md exceeds ~200 lines or contains many discrete instructions packed into a single file. Claude starts ignoring later instructions or misapplying them.

**Why it matters:** Anthropic's documented threshold: under 200 lines. HumanLayer's analysis: Claude Code's own system prompt consumes ~50 of the ~150–200-instruction budget; CLAUDE.md eats the rest. Past the threshold, instruction-following degrades and rules get lost in noise.

**Fix:** Extract older or less-critical content to `agent_docs/<topic>.md` files; reference from CLAUDE.md with `@agent_docs/<topic>.md` pointers. Keep only what's needed *every* session in the main file. Note that imports still load into context — splitting reduces clutter, not tokens.

### CLAUDE-3. Negative framing dominance
**Symptom:** CLAUDE.md is full of `DON'T` statements. Few or no positive instructions about what TO do.

**Why it matters:** Negative instructions tell Claude what to avoid but not what to choose. The space of "not X" is infinite. Claude may avoid X correctly but pick something equally wrong.

**Fix:** Rewrite each `DON'T` as a `DO`. "DON'T use class components" becomes "Use functional components with hooks." Reserve `NEVER` exclusively for genuinely dangerous operations: NEVER delete production data, NEVER commit secrets, NEVER modify files in `<protected_path>`.

### CLAUDE-4. Stale "Current Focus" section
**Symptom:** CLAUDE.md has a "Current Focus" or "What we're working on now" section that describes work completed weeks ago.

**Why it matters:** A stale section is worse than no section. It actively misdirects Claude, who treats CLAUDE.md as authoritative.

**Fix:** Delete the section. That's what `handoff.md` is for — it's overwritten every session and stays current. CLAUDE.md contains only stable context.

### CLAUDE-5. File-system maps that go stale
**Symptom:** CLAUDE.md says "authentication logic lives in `src/auth/handlers.ts`" or includes detailed file-by-file directory listings beyond top-level anchors.

**Why it matters:** "If that file gets renamed or moved, the agent will confidently look in the wrong place" (aihero.dev). Specifics rot the moment the codebase shifts.

**Fix:** Describe capabilities, not paths. "Authentication-related code lives in `src/auth/`" — top-level anchor only. Let Claude generate just-in-time documentation during planning instead.

### CLAUDE-6. Rules a linter already enforces
**Symptom:** CLAUDE.md restates Prettier rules, ESLint rules, formatting rules that `.editorconfig` handles.

**Why it matters:** Duplicated rules drift between CLAUDE.md and CI; CLAUDE.md gets longer for no behavior change. Bijit Ghosh's framing: "If a violation of the rule would block a merge in CI, the rule belongs in CI."

**Fix:** Configure the linter; remove from CLAUDE.md. CLAUDE.md's style section should contain *only* rules that linters can't enforce.

### CLAUDE-7. Secrets in CLAUDE.md
**Symptom:** API keys, connection strings, OAuth client secrets, anything sensitive.

**Why it matters:** Leaks via git history. Even after removal, history retains them.

**Fix:** Move to environment variables and reference by env-var name only. If a secret has already been committed, rotate it and consider repo history rewrite.

### CLAUDE-8. Suggestion-form instructions
**Symptom:** "We generally try to avoid …", "It might be a good idea to …", "Consider doing …".

**Why it matters:** Suggestion-form is ignored by the model. Only imperatives reliably steer behavior.

**Fix:** Rewrite as imperatives. "Avoid X; prefer Y" beats "We generally try to use Y." Be direct.

### CLAUDE-9. Overusing `IMPORTANT` / `YOU MUST`
**Symptom:** Every line emphasized; multiple `IMPORTANT` markers; sentences in all caps throughout.

**Why it matters:** The emphasis loses meaning. When everything is critical, nothing is.

**Fix:** Reserve emphasis for the one or two rules that are genuinely critical. Trust the rest of the doc to be read carefully.

---

<a name="system-level-antipatterns"></a>
## System-Level Anti-Patterns (cross-doc)

### SYS-1. Volatility mixing
**Symptom:** PRD updated mid-sprint with implementation notes; SPEC contains task lists; CLAUDE.md contains the current sprint's plan.

**Why it matters:** Different docs change at different rates. PRD is monthly. SPEC is per-feature. CLAUDE.md is mistake-driven. Task lists are daily. Mixing them means the fastest-changing content rots inside the slower-changing doc.

**Fix:** Keep each doc to its volatility band. Tasks and current-state belong in session-volatile artifacts (handoff.md, plan.md, tasks.md) that Claude Code synthesizes mid-flight, not in PRD/SPEC/CLAUDE.md.

### SYS-2. Docs precede implementation by months
**Symptom:** Detailed plans, sequence diagrams, schemas written for features that haven't started.

**Why it matters:** The plans rot. By the time implementation starts, half the assumptions have shifted. Delete and re-synthesize at implementation time.

**Fix:** Delete the speculative artifacts. Re-synthesize when the work actually starts.

### SYS-3. Parallel files for each AI tool
**Symptom:** Project has CLAUDE.md, `.cursorrules`, `.github/copilot-instructions.md`, AGENTS.md — each maintained separately, slowly diverging.

**Why it matters:** Teams spend time syncing instead of shipping. Divergence becomes its own bug class.

**Fix:** Pick one canonical file. This skill emits CLAUDE.md as the canonical file. If you adopt other tools later, point them at CLAUDE.md (via symlink or `@CLAUDE.md` import) rather than maintaining parallel copies.

### SYS-4. Documents that drift from the code
**Symptom:** PRD describes features that don't exist; SPEC describes an architecture that was abandoned; CLAUDE.md commands don't work.

**Why it matters:** The docs become actively misleading; the team works from chat instead.

**Fix:** Verify commands actually run before declaring CLAUDE.md done. Re-read PRD/SPEC against actual project state monthly. When reality has moved, update the docs. If you can't keep them current, delete them — silence beats wrong.
