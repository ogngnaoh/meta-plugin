---
name: ship-slice
description: Run the slice-ship ritual when a vertical slice is done and ready to mark shipped. Prefer this over doing the steps by hand whenever you're flipping a slice to shipped, closing out a slice, or finishing the last slice of a milestone. It flips status across milestone.md + handoff.md together, freezes the slice file, advances the "Next session start here" pointer, appends the dated ship log line, and lands all of it in one commit on the main-flow branch — so the project's doc state can never end up half-updated. User-invoked only.
disable-model-invocation: true
---

# Ship a slice

The slice-ship ritual in the global `~/.claude/CLAUDE.md` is six edits across three files plus one commit. Done by hand at the end of a long session it's the most-skipped ritual in the workflow, and a partial application is the worst outcome: status flips in `milestone.md` but not `handoff.md`, and a future cold session reads a stale source of truth and trusts it. This skill exists to make the whole thing atomic — all six edits land, or you stop before committing.

The work itself should already be done and verified. This is the *bookkeeping + commit* step, not a place to finish the implementation.

## Step 0 — Confirm this is a conforming project

Check that `docs/milestones.md` exists. If it doesn't, this project doesn't use the milestone/slice convention — stop and say so rather than inventing the structure. (The convention is "synthesize on the fly, never pre-create speculatively.")

## Step 1 — Identify the slice and milestone

From `docs/milestones.md` find the milestone marked `← active`, then read its `docs/NN-milestone/milestone.md` "Slices" index. The shipping slice is normally the one marked `in-progress`.

- If exactly one slice is `in-progress`, that's it — confirm with the user by name before proceeding.
- If it's ambiguous (zero, or more than one in-progress), ask the user which slice is shipping. Don't guess.

Read that slice's `docs/NN-milestone/NN-slice-name.md` and its task checklist.

## Step 2 — Verify it's actually shippable

Shipping is a completion claim, so it needs evidence first: the task checklist is done and the slice's "done" bar (tests, type-check, behavior) was actually *run*, not assumed. If you can't point to that, say so and let the user decide — don't flip status on faith.

If the slice deferred any scope, capture it now in the slice doc's "Out of scope / Deferred" section **and** in any adjacent slice or the milestone's integration-notes section that inherits the deferred work — chat-only deferrals get lost across sessions and worktrees.

## Step 3 — Check the branch (worktree safety)

Run `git rev-parse --is-inside-work-tree`, `git branch --show-current`, and `git rev-parse --show-toplevel`. Determine whether you're in a git worktree (compare the toplevel against the main checkout / `git worktree list`).

The doc edits below **must land on the main-flow branch** — the branch that persists and gets merged. If they only exist on a throwaway worktree branch, they vanish when the worktree is removed. If you're in a worktree or on a branch that won't survive, tell the user where the commit will land and confirm that's the branch they intend, before committing.

## Step 4 — Apply the six edits (do all of them, then commit once)

Make these edits but **do not commit between them** — they're one atomic unit:

1. **`milestone.md` "Slices" list** — flip the shipped slice's status (`planned`/`in-progress` → `shipped`) and advance the next slice's status (`planned` → `in-progress`) if there is one.
2. **`handoff.md` "Slice status across milestone" block** — apply the exact same status flips so the two files agree.
3. **`handoff.md` "Next session start here"** — repoint to the next slice (or the follow-up if this was the milestone's last slice). Make it a concrete literal first action, not "continue work."
4. **Freeze the slice file** — make no further edits to `NN-slice-name.md` after this. (If Step 2 added a deferral note, that edit happens *before* the freeze.)
5. **`handoff.md` ship log** — append a dated line: `- YYYY-MM-DD Slice N (<name>) shipped`. Use today's date from the environment context, not a guessed one.
6. **`handoff.md` "Completed this session"** — record the commit refs of the work that delivered this slice (the implementation commits already on this branch — `git log` the slice's commits). Never try to embed the ship commit's *own* hash here: it doesn't exist until you commit, and a file can't contain the hash of the commit that creates it. The ship commit is the doc commit; reference the already-landed implementation work.

Keep `handoff.md` within ~40 lines. The trick is that one section is append-only and the rest is overwrite-each-session: the **Ship log** is a rolling list you *append* the dated line to, while "Completed this session", "Current state", and "Active concerns" describe only *this* session and get rewritten, not accumulated. If adding the ship line pushes past ~40 lines, prune in this order: (1) stale "Completed this session" detail from prior sessions, (2) the oldest ship-log lines — keep the last 2–3, (3) verbose "Active concerns". Never prune the "Next session start here" pointer or the current "Slice status across milestone" block; git holds the full history of anything you drop.

If this slice was the milestone's last, there's no next slice to advance (edit 1) — instead point "Next session start here" (edit 3) at the milestone follow-up or the next milestone's first slice. Then handle the milestone-ship follow-up: update `docs/milestones.md` to mark this milestone shipped and the next one `← active`. By default this lands in the *same* ship commit — it keeps milestones.md, milestone.md, and handoff.md consistent in one atomic unit. Surface it to the user, and split it into a separate commit only if they explicitly want the milestone close decoupled (e.g. the next milestone isn't decided yet).

## Step 5 — One commit on the main-flow branch

Stage the doc changes (and any remaining slice implementation changes the user wants bundled — surface anything uncommitted via `git status` so nothing is silently left behind), then make a single commit. Suggested message:

```
docs: ship slice NN — <slice name>

Flip status in milestone.md + handoff.md, freeze slice doc,
advance next-session pointer.
```

Confirm the commit landed on the intended (main-flow) branch.

## Step 6 — Report

Tell the user, concisely:
- which slice shipped and which slice is now `in-progress`,
- the commit hash and branch,
- the new "Next session start here" pointer,
- any milestone-ship follow-up still owed.

Then stop. Don't start the next slice unless asked — shipping and starting are separate decisions.
