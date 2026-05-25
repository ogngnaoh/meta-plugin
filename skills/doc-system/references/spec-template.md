# SPEC.md — Canonical Template (full)

This is the long form of the SPEC template. The SKILL.md body has an abridged sketch; this file is the source of truth for what a passing SPEC must contain.

SPEC answers **how** the system will be built: architecture, data model, interfaces, alternatives considered, risks, and validation strategy. The intent (problem, users, goals, non-goals) lives in `PRD.md`; this file picks up where the PRD leaves off.

The single most load-bearing section is **Alternatives Considered**. Industrial Empathy's "A design doc for design docs" calls this "a fixed part of the document structure as it forces the authors to think about what alternatives would have been available." A SPEC without alternatives is an implementation manual, not a design doc.

## Section structure (load-bearing)

1. **Title + status + date** *(required)* — same gating as PRD.
2. **TL;DR (≤5 sentences)** *(required)* — what is being built, the chosen approach, and the single biggest trade-off. Many Google design docs open with this so reviewers who don't need the depth can stop reading early.
3. **Context** — why now, what motivates this. Link to the related PRD.
4. **Goals** *(required, ≥1 numeric)* — what the design must achieve. "'Improve latency' is weak. 'Reduce P95 load time on profile view from 2.1s to <1s' is strong."
5. **Non-goals** *(required, ≥1)* — things that could reasonably be goals but are explicitly chosen not to be. (Distinct from "the system shouldn't crash" — that's not a non-goal, that's a basic correctness expectation.)
6. **Technical context** — language/version, primary dependencies, storage, testing, target platform, scale, constraints. Concrete enough that an implementer doesn't have to guess.
7. **Proposed design** *(required)* — architecture overview (box-and-arrow or prose), data model, interfaces/contracts, key flows.
8. **Alternatives considered** *(required, ≥2 alternatives + do-nothing baseline)* — each alternative + the trade-offs that led to selection. Include the "do nothing" baseline: what happens if we ship nothing; why that's worse. Skipping this baseline is how teams build marginal features that didn't need to exist.
9. **Cross-cutting concerns** *(required — security, privacy, observability each addressed at minimum, even briefly)* — Google's canonical templates require these to be explicitly touched even if briefly because experts in those domains review for them.
10. **Rollout & migration** *(required when applicable)* — phased rollout, feature flags, dark launches, dual writes, kill switch. For greenfield "rollout" may collapse to "deploy".
11. **Testing & validation** *(required)* — unit / integration / e2e coverage; load test plan; how we'll know in prod (dashboards, alerts).
12. **Risks & open questions** — surfaces what could go wrong and what is still undecided. Use `[NEEDS CLARIFICATION: ...]` markers inline elsewhere when local.
13. **Timeline (rough)** — optional; only when it shapes design decisions.

---

## Reusable SPEC template

```markdown
# SPEC: <Feature / System Name>

**Status:** Draft | In Review | Approved | Implemented
**Last updated:** YYYY-MM-DD
**Related PRD:** <link or path>

## 1. TL;DR
<Three to five sentences. What is being built, the chosen approach, the single
biggest trade-off you're accepting.>

## 2. Context
<Why now. What problem motivates this. Pointers to PRD, prior design docs,
any earlier attempts that informed this direction.>

## 3. Goals
- **G1:** <numeric, testable, e.g., "P95 latency <200 ms at 1000 req/s">
- **G2:** <...>

## 4. Non-goals
- **N1:** <explicitly excluded from this design>
- **N2:** <...>

## 5. Technical context
- **Language / version:** <e.g., Python 3.12>
- **Primary dependencies:** <list with versions that matter>
- **Storage:** <e.g., PostgreSQL 16>
- **Testing:** <e.g., pytest + Playwright>
- **Target platform:** <e.g., Linux server, macOS CLI>
- **Performance goals:** <e.g., 1000 req/s, P95 <200 ms>
- **Constraints:** <e.g., offline-capable, <100 MB memory>
- **Scale:** <e.g., 10k DAU, 1M rows/day>

## 6. Proposed design
### 6.1 Architecture overview
<Diagram (ASCII OK) + 1–2 paragraphs explaining the boxes and arrows.>

### 6.2 Data model
<Entities, key fields, relationships, invariants. SQL-style schema OK.>

### 6.3 Interfaces / contracts
<API endpoints, CLI surfaces, message schemas, file formats.>

### 6.4 Key flows
<Sequence walks for the 2–3 most important paths. ASCII sequences are fine.>

## 7. Alternatives considered
### Alt A: <Name>
- **What it is:** <one paragraph>
- **Why rejected:** <trade-off vs. chosen design>

### Alt B: <Name>
- **What it is:** <one paragraph>
- **Why rejected:** <trade-off vs. chosen design>

### "Do nothing" baseline
- **What happens if we ship nothing:** <one paragraph>
- **Why that's worse:** <what gets worse, for whom, by how much>

## 8. Cross-cutting concerns
- **Security:** <auth model, secrets handling, blast radius if compromised>
- **Privacy:** <PII handling, retention, deletion path>
- **Observability:** <logs, metrics, traces, SLOs you'll define>
- **Accessibility / i18n (where UI exists):** <key considerations>

## 9. Rollout & migration
- **Phase 1:** <what ships first>
- **Phase 2:** <what ships next>
- **Feature flags / dark launch / kill switch:** <which apply>
- **Data migration plan (if any):** <forward/backward compatibility>

## 10. Testing & validation
- **Unit / integration / e2e:** <coverage targets, where each lives>
- **Load test plan:** <what you'll run before launch>
- **How we'll know in prod:** <dashboards, alerts, SLO breach response>

## 11. Risks & open questions
- **Risk:** <description> — **Mitigation:** <how you'll handle it>
- [NEEDS CLARIFICATION: <cross-cutting unknown>]
```

---

## Filled example (solo-dev, terse)

```markdown
# SPEC: md2email

**Status:** Draft
**Last updated:** 2026-05-25
**Related PRD:** ./PRD.md

## 1. TL;DR
md2email is a Python 3.12 CLI that converts a Markdown file to email-safe
HTML by parsing with `markdown-it-py`, running the output through a
preserve-walk that inlines a hardcoded stylesheet, and writing to a
`.html` file beside the input. The single biggest trade-off accepted:
hardcoded styles for v1, no `--css` flag, in exchange for a one-binary
zero-config tool that just works.

## 2. Context
Per PRD.md, current per-issue formatting cost is 20 min. v1 is a CLI
because the workflow is "save file, run command, paste HTML" — no
server or GUI needed.

## 3. Goals
- **G1:** Convert a 5k-word file in <2 seconds on a 2020 MacBook Air.
- **G2:** Output renders identically (visual spot-check) in Gmail web,
  Outlook 2021, and Apple Mail on macOS 15.

## 4. Non-goals
- **N1:** No interactive REPL or watch mode in v1.
- **N2:** No bundled image hosting / optimization.

## 5. Technical context
- **Language:** Python 3.12 (uv-managed venv)
- **Primary deps:** click 8.x (CLI), markdown-it-py 3.x (parser),
  premailer 3.x (CSS-to-inline transformer)
- **Storage:** None (file-in / file-out)
- **Testing:** pytest + pytest-snapshot for HTML diffs
- **Target platform:** macOS 14+, Linux (Ubuntu 22.04+); Windows best-effort
- **Performance goals:** <2s on 5k-word file
- **Constraints:** Pure-Python deps only (no Rust toolchain for users)

## 6. Proposed design
### 6.1 Architecture overview
Single-process pipeline:

```
input.md  →  markdown-it-py.parse  →  HTML AST
                                          ↓
                          apply_email_safe_transforms()
                                          ↓
                              premailer.transform()
                                          ↓
                                    write(output.html)
```

### 6.2 Data model
No persistent storage. In-memory: HTML AST (markdown-it tokens), an
intermediate full HTML string, and the inlined output string.

### 6.3 Interfaces / contracts
- `md2email convert <input.md>` — converts in place, writes `<input>.html`
- `md2email convert <input.md> --output <out.html>`
- `md2email convert <input.md> --force` — overwrite existing output
- Exit codes: 0 ok, 1 input-error, 2 internal-error

### 6.4 Key flows
1. **Happy path:** parse → transform → inline → write → exit 0.
2. **Output exists:** check exists → if no --force, exit 1 with message.
3. **Malformed Markdown:** markdown-it raises → wrap → stderr → exit 1.

## 7. Alternatives considered
### Alt A: Pure markdown-it with custom email-CSS plugin
- **What it is:** Write a markdown-it plugin that emits email-safe HTML
  directly, no inlining step.
- **Why rejected:** Premailer already handles all the email-client
  quirks; rewriting that logic in a plugin is 200+ lines of CSS-table
  munging we'd own forever.

### Alt B: Use `mistletoe` instead of `markdown-it-py`
- **What it is:** mistletoe is pure-Python, simpler, and faster on
  small inputs.
- **Why rejected:** Smaller plugin ecosystem; markdown-it-py is
  CommonMark-strict which matches Obsidian's behavior more closely,
  reducing surprises.

### "Do nothing" baseline
- **What happens if we ship nothing:** I keep losing 20 min per issue
  manually fixing tables and inlining styles in Mailchimp's editor.
  Annoyance compounds across ~50 issues/year (~16 hours/year).
- **Why that's worse:** This is exactly the kind of toil that's worth
  a weekend of build time.

## 8. Cross-cutting concerns
- **Security:** No network calls. No code execution from input. Premailer's
  CSS parser is the largest attack surface; track its CVEs.
- **Privacy:** Tool runs locally, reads/writes only the paths the user
  specifies. No telemetry.
- **Observability:** stderr for warnings; `--verbose` flag enables debug
  logging to stderr. No remote metrics.

## 9. Rollout & migration
- **Phase 1:** Local install via `uv tool install md2email`.
- **Phase 2:** PyPI publish after 4 weeks of dogfooding.
- **Kill switch:** N/A (CLI, user controls when to upgrade).

## 10. Testing & validation
- **Unit:** parser-output snapshots for every Markdown construct.
- **Integration:** full-pipeline snapshots for a corpus of 5 representative
  newsletter issues.
- **Manual:** weekly send → visual spot-check in Gmail/Outlook/Apple Mail.

## 11. Risks & open questions
- **Risk:** premailer maintainership has slowed in 2025 — **Mitigation:**
  pin major version; revisit alt B if a critical CVE lands unpatched.
- [NEEDS CLARIFICATION: do we want a `--watch` flag in v1 or defer to v2?]
```

---

## Discipline notes (what makes a SPEC pass the gate)

- **TL;DR up front.** Readers stop after the first sentence. Carry the load there.
- **Numeric goals.** "Reduce latency" is weak; "Reduce P95 from 2.1s to <1s" is strong.
- **Two alternatives plus the do-nothing baseline.** Always. Even for solo work. Without alternatives, you can't show you picked the best of a real set — and you'll discover the alternative the hard way after merge.
- **Cross-cutting concerns get one line minimum.** Security, privacy, observability. If a section truly doesn't apply, write "N/A — <one-line reason>" so the gate sees it was considered, not skipped.
- **Length is bounded by completeness, not pages.** The longer the doc, the harder to consume and change. As short as possible, as long as necessary. If you're past 4 pages and still not at testing, ask: is this a SPEC, or is it really an implementation manual?
- **Reviewable as a PR.** The SPEC lives in the repo. Treat it like code: review it on commits, prune it monthly.
