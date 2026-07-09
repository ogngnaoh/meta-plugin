---
name: ship-slice
description: Run the slice-ship ritual when a vertical slice is done and ready to mark shipped. Prefer this over doing the steps by hand whenever you're flipping a slice to shipped, closing out a slice, or finishing the last slice of a milestone. It flips slice status in milestone.md (the single live home of status), advances handoff.md's "Next session start here" pointer, shuts down any teammates or standing goals the slice ran, and lands the doc edits in one commit on the main-flow branch — so the project's doc state can never end up half-updated. User-invoked only.
disable-model-invocation: true
---

# Ship a slice

The slice-ship ritual is two doc edits plus machinery shutdown plus one commit. Done by hand at the end of a long session it's the most-skipped ritual in the workflow, and a partial application is the worst outcome: status flips in `milestone.md` but the handoff pointer goes stale, and a future cold session reads a stale source of truth and trusts it. **This skill is the ritual's single home** — the global `~/.claude/CLAUDE.md` points here rather than restating the steps — and it exists to make the whole thing atomic: all edits land, or you stop before committing.

> Reconciled 2026-07-09 against the rewritten global CLAUDE.md: slice status lives ONLY in `milestone.md`; `handoff.md` is overwrite-only (three sections, ≤40 lines, no status tables, no ship log, no commit refs — git holds history). Shelfware re-check ~2026-08-09: if this skill still isn't being invoked at ship time by then, delete it and fold the ritual back into prose (routing guide §6 sunset rule).

The work itself should already be done and verified. This is the *bookkeeping + shutdown + commit* step, not a place to finish the implementation.

## Step 0 — Confirm this is a conforming project

Check that `docs/milestones.md` exists. If it doesn't, this project doesn't use the milestone/slice convention — stop and say so rather than inventing the structure. (The convention is "synthesize on the fly, never pre-create speculatively.")

## Step 1 — Identify the slice and milestone

From `docs/milestones.md` find the milestone marked `← active`, then read its `docs/NN-milestone/milestone.md` "Slices" index. The shipping slice is normally the one marked `in-progress`.

- If exactly one slice is `in-progress`, that's it — confirm with the user by name before proceeding.
- If it's ambiguous (zero, or more than one in-progress), ask the user which slice is shipping. Don't guess.

Read that slice's working doc and its task checklist.

## Step 2 — Verify it's actually shippable

Shipping is a completion claim, so it needs evidence first: the task checklist is done and the slice's "done" bar (tests, type-check, behavior) was actually *run*, not assumed. If the verification was produced entirely by the same span that wrote the code, say so — aggregate green from the machinery that wrote both the code and its tests is a status report, not verification. If you can't point to evidence, say so and let the user decide — don't flip status on faith.

If the slice deferred any scope, capture it now in the slice's working doc **and** in the `milestone.md` integration-notes section (or the adjacent slice that inherits it) — chat-only deferrals get lost across sessions and worktrees.

## Step 3 — Check the branch (worktree safety)

Run `git rev-parse --is-inside-work-tree`, `git branch --show-current`, and `git rev-parse --show-toplevel`. Determine whether you're in a git worktree (compare the toplevel against the main checkout / `git worktree list`).

The doc edits below **must land on the main-flow branch** — the branch that persists and gets merged. If they only exist on a throwaway worktree branch, they vanish when the worktree is removed. If you're in a worktree or on a branch that won't survive, tell the user where the commit will land and confirm that's the branch they intend, before committing.

## Step 4 — Apply the two doc edits (both, then commit once)

Make these edits but **do not commit between them** — they're one atomic unit:

1. **`milestone.md` "Slices" list** — flip the shipped slice's status (`planned`/`in-progress` → `shipped`) and advance the next slice's status (`planned` → `in-progress`) if there is one. This is the ONLY place slice status lives.
2. **`handoff.md`** — overwrite to current reality, three sections only: repoint **Next session start here** to the next slice's concrete literal first action (not "continue work"); refresh **Current state** and **Active concerns** to describe now. Keep it ≤40 lines. No status tables, no ship log, no commit refs — `milestone.md` and git hold those.

If this slice was the milestone's last, there's no next slice to advance — instead point "Next session start here" at the milestone follow-up or the next milestone's first slice, and handle the milestone close: update `docs/milestones.md` to mark this milestone shipped and the next one `← active`. By default this lands in the *same* ship commit — it keeps milestones.md, milestone.md, and handoff.md consistent in one atomic unit. Surface it to the user, and split it into a separate commit only if they explicitly want the milestone close decoupled.

## Step 5 — Shut down the slice's machinery

An idle teammate keeps burning and a stale standing goal fights the next pivot, so shutdown is part of shipping, not an afterthought:

- Shut down any teammates, panes, or fleets the slice ran.
- Retire or re-scope any standing goal / Stop hook that was set for this slice — its exit criteria are now met or moot.
- Remove worktrees whose branches have landed (`git worktree remove`), after confirming the doc edits above are on the main-flow branch, not in the worktree.

## Step 6 — One commit on the main-flow branch

Stage the doc changes (and any remaining slice implementation changes the user wants bundled — surface anything uncommitted via `git status` so nothing is silently left behind), then make a single commit. Suggested message:

```
docs: ship slice NN — <slice name>

Flip status in milestone.md, advance handoff next-session pointer.
```

Confirm the commit landed on the intended (main-flow) branch.

## Step 7 — Report

Tell the user, concisely:
- which slice shipped and which slice is now `in-progress`,
- the commit hash and branch,
- the new "Next session start here" pointer,
- what machinery was shut down (teammates, goals, worktrees),
- any milestone-ship follow-up still owed.

Then stop. Don't start the next slice unless asked — shipping and starting are separate decisions.
