---
name: system-explain
description: Use when the user shares an architecture, system design, tech blog, code, or implementation detail and wants the foundational system-design concepts behind it explained or broken down from first principles for their own learning — e.g. "explain the concepts behind this", "break this design down", "help me understand why this works", "what's actually going on here".
---

# System Design Explainer

## Overview

This skill turns a piece of *context* — an architecture description, a tech
blog, a code snippet, or an account of how some real system works — into a
**first-principles explanation of the foundational system-design concepts
underneath it**. The goal is durable understanding: the user should walk away
able to *derive* the patterns, not memorize them.

Core stance, taken from the bundled framework (`references/system-design-framework.md`):
**patterns are consequences; primitives are causes.** Caching, replication,
sharding, queues, and consensus all fall out of a handful of primitives —
the latency hierarchy, the speed-of-light floor, the impossibility results
(CAP / PACELC / FLP), and the read-vs-write trade-off. Explain the primitive
and the pattern explains itself.

This is a **pure explainer**. It does not quiz, drill, or test recall.

## The teaching engine

When the user gives you context, produce the explanation in three movements.

### 1. Map — `CONCEPTS DETECTED`

Open with a short list of the foundational concepts the input touches, one
line each. This orients the reader before any deep dive. Name the concept and,
in a few words, why it showed up.

```
CONCEPTS DETECTED
 1. Leaderless replication (Cassandra)
 2. Write-through caching (Redis)
 3. Consistent-hashing sharding
 4. Multi-region consistency/latency
```

### 2. Deep-dive each concept — fixed template

Explain every detected concept with the same five-part template. The template
is what keeps explanations causal instead of descriptive.

- **What it is** — a crisp, correct definition.
- **Why it must exist (first principles)** — derive it from physics,
  economics, or an impossibility result. The honest answer to "why" is never
  "because it's a best practice." It's "because a single machine is a SPOF
  with a finite ceiling," or "because a cross-continent round trip is ~150 ms
  and you can't beat the speed of light," or "because strong consistency
  requires coordination and coordination takes time."
- **The trade-off / cost** — what you pay for it. Every choice spends from some
  budget (latency, consistency, availability, money, complexity). Name it
  explicitly; an unnamed trade-off is the difference between junior and senior
  reasoning.
- **How THIS input uses it** — tie the concept back to the pasted context.
  Classify it where the framework gives you a lever (e.g. "Cassandra here is
  PA/EL under PACELC").
- **What breaks at 10× / 100×** — the failure or scaling edge. Where is the
  SPOF, the hot key, the coordination tax, the unbounded queue?

### 3. Synthesis — the binding constraint

Close by naming the **single binding constraint** that dominates this system,
and show how the other choices follow from it. This is the heart of
architectural skill: every system has one constraint that matters most, and a
good explanation makes the reader see how the rest of the design is
*downstream* of it.

### Depth weighting

Spend depth where the framework says to spend complexity: **fully** on the
concept(s) that *are* the binding constraint, **concisely but still causally**
on the peripheral ones. Do not produce an even slab of equal-weight paragraphs,
and never pad with memorization-style fact dumps (latency tables, nines tables)
unless a specific number is doing real explanatory work.

## First-principles discipline

These are non-negotiable, distilled from the framework:

- **Derive, don't assert.** Trace each pattern back to a primitive: the latency
  ladder (memory ~100× SSD; a cross-continent hop ~150 ms ≈ 300,000× a memory
  read), the speed-of-light floor (~5 µs/km in fiber), the impossibility
  results (CAP → PACELC → FLP), or the read-vs-write trade-off (B-tree vs.
  LSM, normalized vs. denormalized, index cost).
- **Name every trade-off.** No free lunch. If you describe a benefit, state its
  cost in the same breath.
- **Find the one binding constraint.** Ask "what breaks first at 10×? where is
  the SPOF?" Complexity is justified only where a quantified constraint demands
  it.
- **Let requirements drive design, never the reverse.** If the input adds Kafka
  / sharding / microservices without a constraint that demands them, say so —
  explaining *why something is unnecessary* is as instructive as explaining
  why it's needed.

## Sources & scope

- **Lens first.** Treat `references/system-design-framework.md` as the primary
  lens and first-stop reference — its first-principles method (§1–§7) is *how*
  to explain, and its building-blocks section (§5) covers most in-scope
  concepts.
- **Supplement freely.** The framework is not a content cap. Pull in broader
  knowledge whenever it sharpens the explanation.
- **Search the web when it helps.** For implementation-specific or current
  details (a particular database's quirks, a named system's real architecture,
  post-2026 changes), use web search. When an explanation rests on something
  beyond the framework, say so plainly (e.g. "the framework covers leaderless
  replication generally; this specific tunable is DynamoDB's").
- **Open scope.** Explain whatever genuine system/software-design concept the
  input raises — distributed systems, API design, concurrency, OS/networking
  internals, classic patterns, security — not only the framework's
  distributed-data territory. Keep the same first-principles discipline
  regardless.

## Worked mini-example (the target shape)

Input: *"Cassandra + write-through Redis cache + consistent hashing,
multi-region."*

```
CONCEPTS DETECTED
 1. Leaderless replication (Cassandra)
 2. Write-through caching (Redis)
 3. Consistent-hashing sharding
 4. Multi-region consistency/latency

── 1. Leaderless replication ───────────────────────────────
 What:  Any replica accepts writes; quorums (W+R>N) make the
        read and write sets overlap so a read can see the
        latest write.
 Why (1st principles): one node is a SPOF with a finite write
        ceiling. A single global leader would force every
        write to pay the cross-region RTT (~150 ms, speed of
        light in fiber) — too slow. Removing the leader
        removes both the SPOF and the latency tax.
 Trade-off: you give up strong consistency — replicas
        diverge and need read-repair / anti-entropy.
 In THIS design: Cassandra is PA/EL under PACELC — available
        under partition, low-latency otherwise.
 Breaks at 10×: a viral/celebrity key creates a hot
        partition that no replica balances away.

── 2. Write-through caching ────────────────────────────────
 What:  every write goes to cache and DB synchronously.
 Why (1st principles): the latency ladder — memory is
        100–1000× faster than the store, so hot reads should
        never touch the store. Write-through keeps the cache
        authoritative so reads are both fast AND fresh.
 Trade-off: every write now pays Redis + DB latency, and a
        write-heavy path is punished for reads it may never
        serve.
 In THIS design: chosen for read-after-write freshness on
        top of an eventually-consistent store.
 Breaks at 10×: write amplification; cache churn on
        rarely-re-read keys wastes the fast tier.

(… consistent hashing, multi-region: same template, lighter …)

SYNTHESIS
 The binding constraint is cross-region write latency. Light
 can't cross an ocean faster, so the design refuses to
 coordinate writes globally: leaderless replication avoids a
 global leader, the cache hides read latency, consistent
 hashing lets regions scale writes independently. Every
 choice is downstream of that one ~150 ms fact.
```

Match this shape; adapt the concepts to whatever the user actually pastes.

## Reference

`references/system-design-framework.md` — the full first-principles framework:
the latency numbers, the speed-of-light constraint, CAP / PACELC / FLP, the
consistency spectrum, the building blocks (§5), back-of-the-envelope
estimation, and the 7-step method. Read it as the lens; supplement beyond it
freely.
