# PRD.md — Canonical Template (full)

This is the long form of the PRD template. The SKILL.md body has an abridged sketch; this file is the source of truth for what a passing PRD must contain.

A PRD answers **what** is being built and **why**, **for whom**, and **how we'll know it worked** — explicitly *not* how the system will implement it. Implementation belongs in `SPEC.md`. If you find yourself naming frameworks, databases, or specific APIs, stop and move them.

## Section structure (load-bearing)

Every PRD this skill emits has these sections in this order. Sections marked **required** are enforced by the quality gate (`quality-gate.md`); skipping one means the PRD won't be emitted until you either fill it or insert `[NEEDS CLARIFICATION: ...]` markers that explain why it's deferred.

1. **Title + status + date** *(required)* — owner is "me" by default for solo work, but date and status (`Draft | In Progress | Approved`) matter because the PRD is a living artifact.
2. **Problem** *(required)* — one paragraph. Who has this pain, when, and why current solutions fail. Lenny Rachitsky's framing: "Nailing the problem statement is the single most important step in solving any problem."
3. **Target users** *(required)* — ≥1 persona. For solo projects "me" is valid, but be specific about *which version* of you ("me when I'm triaging incidents at 2am" beats "me").
4. **Goals & success metrics** *(required)* — ≥1 measurable goal. Replace "improve performance" with concrete numeric targets like "reduce P95 page-load on profile view from 2.1s to <1s". Missing measurable metrics is the most common rookie PRD mistake.
5. **Non-goals** *(required, ≥3)* — explicit out-of-scope items. The single highest-leverage section for preventing scope creep; every elite PRD template surveyed (Intercom, Asana, Shape Up, Lenny's 1-Pager) carries this section.
6. **User stories with acceptance scenarios** *(required, ≥1 story)* — Given/When/Then format. Translates goals into behaviors a test can verify.
7. **Functional requirements** *(required, ≥1 FR)* — `FR-NNN` zero-padded numbering, written as "System MUST …" lines. Testable, deterministic, **technology-agnostic**.
8. **Non-functional requirements / constraints** — performance, security, privacy, accessibility — the qualities the build must hit. Optional but commonly present.
9. **Edge cases** — boundary conditions and error scenarios. Spec Kit treats this as a mandatory companion to user stories.
10. **Assumptions & dependencies** — what the PRD takes for granted about users, environment, or other teams.
11. **Open questions** — anything that didn't get resolved during the interview. Use `[NEEDS CLARIFICATION: ...]` markers at the point of impact (inline in the FR or story), not buried here, when the ambiguity is local. Use this section for cross-cutting unknowns.

---

## Reusable PRD template

```markdown
# PRD: <Feature / Product Name>

**Status:** Draft | In Progress | Approved
**Last updated:** YYYY-MM-DD

## 1. Problem
<One paragraph: who has this pain, when, and why current solutions fail.>
<If relevant, link to: prior support tickets, analytics, competitive analysis,
or your own past frustration logs.>

## 2. Target users
- **Primary:** <role, context, what they're trying to do>
- **Secondary (optional):** <role>

## 3. Goals & success metrics
- **G1:** <verb + outcome>
  - **Metric:** <quantitative, time-bounded, e.g., "P1 task completion >90% within 30 days">
- **G2:** <...>
  - **Metric:** <...>

## 4. Non-goals (out of scope)
- N1: <explicitly NOT in this release>
- N2: <deferred to a later release>
- N3: <a thing reviewers might assume but shouldn't>

## 5. User stories
### Story 1 — <Title>
**As a** <persona> **I want** <capability> **so that** <benefit>.

**Acceptance scenarios:**
- Given <state>, when <action>, then <outcome>.
- Given <state>, when <action>, then <outcome>.

### Story 2 — <Title>
<...>

## 6. Functional requirements
- **FR-001:** System MUST <capability>.
- **FR-002:** Users MUST be able to <interaction>.
- **FR-003:** System MUST <data behavior> [NEEDS CLARIFICATION: <what's missing>].

## 7. Non-functional requirements
- **Performance:** <e.g., P95 latency <200 ms under expected load>
- **Security / privacy:** <e.g., no PII in logs; tokens encrypted at rest>
- **Accessibility:** <e.g., WCAG 2.2 AA where UI exists>
- **Scale:** <e.g., 10k DAU at launch>

## 8. Edge cases
- What happens when <boundary condition>?
- How does the system handle <expected error>?
- What happens at empty / zero / max-input?

## 9. Assumptions & dependencies
- **Assume:** <thing you're treating as given>
- **Depend on:** <external system, vendor, approval, person>

## 10. Open questions
- [NEEDS CLARIFICATION: cross-cutting ambiguity that didn't fit inside a single FR or story]
```

---

## Filled example (solo-dev)

This is the PRD shape the skill emits for a small CLI project. Keep it as a reference for tone and density.

```markdown
# PRD: md2email

**Status:** Draft
**Last updated:** 2026-05-25

## 1. Problem
Newsletter authors who write in Markdown spend ~20 minutes per issue manually
reformatting their content for email — inlining styles, fixing table layouts,
shrinking images — because the standard Markdown-to-HTML pipelines produce
output that breaks in major email clients (Gmail, Outlook, Apple Mail).

## 2. Target users
- **Primary:** Me, writing my weekly newsletter. I author in Obsidian, paste
  the export into Mailchimp, and currently lose 20 minutes per send fixing layout.
- **Secondary:** Future open-source users with the same workflow.

## 3. Goals & success metrics
- **G1:** Reduce per-issue formatting time from 20 min to under 2 min.
  - **Metric:** Self-reported elapsed time across 4 consecutive issues.
- **G2:** Output renders correctly in Gmail, Outlook 2021, Apple Mail.
  - **Metric:** Manual visual check across the three clients per issue, with
    zero layout regressions for 4 consecutive issues.

## 4. Non-goals
- N1: WYSIWYG editor — input is always Markdown files.
- N2: Sending email — this tool only produces HTML.
- N3: Image hosting — users supply hosted image URLs.

## 5. User stories
### Story 1 — Convert a Markdown file to email HTML
**As a** newsletter author **I want** to run a single command on my Markdown
file **so that** I get email-safe HTML I can paste into my email tool.

**Acceptance scenarios:**
- Given a Markdown file with headings, paragraphs, links, and one table,
  when I run `md2email convert input.md`, then I get `input.html` with
  inlined styles that render identically in Gmail, Outlook, and Apple Mail.
- Given a Markdown file containing an inline image reference,
  when I run `md2email convert input.md`,
  then the output preserves the image with appropriate `max-width` styling.

## 6. Functional requirements
- **FR-001:** System MUST accept a Markdown file path as a positional argument.
- **FR-002:** System MUST emit an `.html` file next to the input by default,
  or to a path supplied via `--output`.
- **FR-003:** System MUST inline all styles (no `<style>` blocks, no external
  CSS references) [NEEDS CLARIFICATION: should I support a `--css` flag to
  inject a user stylesheet, or hardcode the rules for v1?].
- **FR-004:** System MUST preserve link text and href and produce `<a>` tags
  with target="_blank".
- **FR-005:** System MUST output tables with email-safe attributes
  (`cellpadding`, `cellspacing`, `border`).

## 7. Non-functional requirements
- **Performance:** Process a 5k-word Markdown file in under 2 seconds.
- **Portability:** Run on macOS and Linux; Windows is best-effort.

## 8. Edge cases
- What happens with a 0-byte input file? → emit an empty `<body>`, exit 0.
- What happens when the output path already exists? → require `--force` to overwrite.
- What happens with malformed Markdown? → fail with a clear stderr message, exit 1.

## 9. Assumptions & dependencies
- **Assume:** Authors host their own images and reference them by URL.
- **Depend on:** A maintained Markdown parser; will pick one in SPEC.

## 10. Open questions
- [NEEDS CLARIFICATION: do we need a "preview in browser" subcommand for v1
  or can that be a later release?]
```

---

## Discipline notes (what makes a PRD pass the gate)

- **Tech-agnostic.** No framework names, no library names, no database choices, no API SDKs. Those are SPEC concerns. The PRD says *what the user experiences*; SPEC says *how the system delivers it*. (Spec Kit Discussion #980 makes this an explicit blocker for merge.)
- **Metrics are numeric.** "Improve UX" or "make it faster" is not a metric. "Reduce P95 from 2.1s to <1s" is.
- **At least 3 non-goals.** This is non-negotiable. Less than 3 means you haven't actually thought about scope.
- **Each FR is independently testable.** Read each FR and ask: "Could someone write a test for this without asking me follow-up questions?" If no, refine it or mark it `[NEEDS CLARIFICATION: ...]`.
- **`[NEEDS CLARIFICATION: ...]` is honest, not a failure.** Inline these at the point of impact. They make ambiguity visible at the moment it can be resolved. A PRD with several inline `[NEEDS CLARIFICATION]` markers is more honest than a PRD with none and silent assumptions everywhere.
- **Living document, but timestamped.** Update the date when the PRD changes materially. Add a one-line note at the bottom under `## Change log` if you want to track what shifted.
