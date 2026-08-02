---
description: Run the four-agent feature pipeline — plan, implement, test, review — halting at anything a human needs to decide.
argument-hint: <feature request>
---

Run the full feature pipeline for: **$ARGUMENTS**

If `$ARGUMENTS` is empty, ask what to build and stop.

You are the orchestrator. You do not plan, write, test, or review anything yourself —
you delegate each stage to its subagent and gate the handoffs. Run the stages strictly
in order. Never skip ahead, and never start a stage before the previous stage's handoff
file exists on disk.

## Stage 0 — Preflight

1. Confirm this is a git repo. If not, stop and say so.
2. Check the current branch. If it is `main`, `master`, or the repo's default branch,
   **stop** and ask whether to create a feature branch first. Do not create one
   unprompted.
3. Report any uncommitted changes and confirm before proceeding — the Reviewer diffs
   the working tree, and pre-existing changes will muddy the verdict.
4. Clear stale handoffs: delete `.pipeline/spec.md`, `.pipeline/changes.md`,
   `.pipeline/test-results.md`, and `.pipeline/review.md` if they exist. Create
   `.pipeline/` if needed. Stale files from a previous run are the most common way
   this pipeline goes wrong.
5. If `.pipeline/` is not covered by `.gitignore`, add it and say that you did.

## Stage 1 — Plan

Delegate to the **pipeline-planner** subagent with the feature request above.

Wait for `.pipeline/spec.md`.

**Gate:** If the spec has a non-empty "Open questions" section, **stop the pipeline**.
Show the questions verbatim and ask the human to answer them. Do not answer them
yourself, and do not proceed on a best guess.

## Stage 2 — Implement

Delegate to the **pipeline-coder** subagent. Tell it to read `.pipeline/spec.md`.

Wait for `.pipeline/changes.md`.

**Gate:** If the Coder reported that it stopped, or that the build failed, stop and
show the human why.

## Stage 3 — Test

Delegate to the **pipeline-tester** subagent.

Wait for `.pipeline/test-results.md`.

**Gate:** If the verdict is FAIL, **stop the pipeline**. Show the failures and the
Tester's diagnosis. Do not fix the code yourself, do not re-run the Coder, and do not
proceed to review. A human decides what happens next.

## Stage 4 — Review

Delegate to the **pipeline-reviewer** subagent.

Wait for `.pipeline/review.md`. Show it to the human in full.

## Report

End with a short summary: the verdict, the files touched, the test totals, and the
single most important thing for the human to look at.

**Do not merge, push, commit, or tag anything.** Leave the branch as it is. The human
is the final gate — that is the whole arrangement.
