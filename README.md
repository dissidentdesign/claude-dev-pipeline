<div align="center">

```
██████╗ ██╗██████╗ ███████╗██╗     ██╗███╗   ██╗███████╗
██╔══██╗██║██╔══██╗██╔════╝██║     ██║████╗  ██║██╔════╝
██████╔╝██║██████╔╝█████╗  ██║     ██║██╔██╗ ██║█████╗  
██╔═══╝ ██║██╔═══╝ ██╔══╝  ██║     ██║██║╚██╗██║██╔══╝  
██║     ██║██║     ███████╗███████╗██║██║ ╚████║███████╗
╚═╝     ╚═╝╚═╝     ╚══════╝╚══════╝╚═╝╚═╝  ╚═══╝╚══════╝

        p l a n  ·  b u i l d  ·  t e s t  ·  j u d g e
```

**four agents · one command · you approve the merge**

Pipeline design by **[Ray Fu](https://www.skool.com/@ray-fu-5306)** · packaged as an installable Claude Code plugin

</div>

```
   ╔═══════════╗   ╔═══════════╗   ╔═══════════╗   ╔═══════════╗
   ║  PLANNER  ║──▶║   CODER   ║──▶║  TESTER   ║──▶║ REVIEWER  ║
   ║   opus    ║   ║  sonnet   ║   ║  sonnet   ║   ║   opus    ║
   ╚═════╤═════╝   ╚═════╤═════╝   ╚═════╤═════╝   ╚═════╤═════╝
         │               │               │               │
      spec.md       changes.md    test-results.md    review.md
         │               │               │               │
         ▼               ▼               ▼               ▼
   open questions?   build broke?    tests red?      NEEDS WORK?
         └───────────────┴───────┬───────┴───────────────┘
                                 │
                                 ▼
                     [ HALT — human decides ]
```

A four-agent feature pipeline for [Claude Code](https://claude.com/claude-code).

One command — `/pipeline` — runs a **Planner**, a **Coder**, a **Tester**, and a
read-only **Reviewer** in sequence. Each agent hands off to the next through a file in
`.pipeline/`, and the run stops at the first thing a human actually needs to decide.

```
/pipeline add rate limiting to the login endpoint, 5 attempts per minute per IP, return 429
```

Nothing is committed, pushed, or merged. You get a branch and a verdict.

---

## Install

From your shell — two commands, run one at a time:

```bash
claude plugin marketplace add dissidentdesign/claude-dev-pipeline
```

```bash
claude plugin install dev-pipeline@claude-dev-pipeline
```

**Restart Claude Code.** Plugins are loaded at session start, so `/pipeline` won't
appear in a session that was already open.

That's it — `/pipeline` is now available in **every** project on this machine. No
per-project files to copy.

<details>
<summary>Installing from inside Claude Code instead</summary>

The same two steps work as slash commands, but they must be run **separately** —
`/plugin marketplace add` opens a dialog and waits for input:

```
/plugin marketplace add
```

When it prompts for a marketplace source, enter **only** this, with nothing appended:

```
dissidentdesign/claude-dev-pipeline
```

Then, as a separate command:

```
/plugin install dev-pipeline
```

Pasting both lines into the source prompt at once produces
`'... /plugin install dev-pipeline' is not a valid GitHub owner/repo shorthand` —
the whole string was read as the repo name.

</details>

To update later:

```bash
claude plugin marketplace update claude-dev-pipeline
```

> If another plugin or skill also defines `pipeline`, invoke this one explicitly as
> `/dev-pipeline:pipeline`.

---

## Why four agents instead of one

A single agent asked to plan, implement, test, and review holds all four jobs in one
context window. The plan, the code, the test output, and the critique compete for the
same attention, and the agent ends up grading its own homework with the answer key
still open.

Splitting the work buys three things:

**Clean context per role.** The Coder reads a spec, not a transcript of how the spec
was argued into existence. The Tester reads what changed, not why. Each agent's window
holds only what its job needs.

**Enforced handoffs.** Agents communicate through files, not conversation. That makes
every stage inspectable — you can read exactly what the Coder was told and exactly
what it claims it did — and it makes the boundaries real rather than aspirational.

**Structural honesty.** The Tester cannot fix the code it tests. The Reviewer cannot
edit the code it judges. Neither restriction is a matter of the agent choosing to
behave; the tools simply aren't there. An agent that *could* quietly patch a problem
is measurably more inclined to patch it than report it.

## The agents

| Agent | Model | Tools | Output |
|-------|-------|-------|--------|
| `pipeline-planner` | Opus | Read, Grep, Glob, Write | `spec.md` — files, signatures, edge cases, patterns to follow, explicit out-of-scope |
| `pipeline-coder` | Sonnet | Read, Write, Edit, Grep, Glob, Bash | the actual code, plus `changes.md` |
| `pipeline-tester` | Sonnet | Read, Write, Edit, Grep, Glob, Bash | test files, plus `test-results.md` |
| `pipeline-reviewer` | Opus | Read, Grep, Glob, Bash, Write | `review.md` with a verdict |

**Opus for planning and review, Sonnet for implementation and testing.** Planning sets
the ceiling — a vague spec produces vague code no matter how strong the implementer —
and review is the last chance to catch something before it reaches you. Both run once
per feature. Implementation and testing produce far more tokens but are bounded,
well-specified work, which is where Sonnet is strongest per dollar. The practical
effect is that the majority of token spend lands on the cheaper model without the
expensive reasoning steps being cheapened.

## Where the pipeline stops

By design, it stops often. Every halt is a place where continuing would mean an agent
inventing an answer:

- **Open questions in the spec.** The Planner flags anything underspecified enough to
  change the implementation rather than guessing. You answer, then re-run.
- **Build failure.** The Coder reports rather than thrashing.
- **Failing tests.** The Tester diagnoses and stops. It does not touch the
  implementation — if it did, it would be marking its own work.
- **A non-SHIP verdict.** You decide whether to fix by hand or re-run with a sharper
  request.

## Overnight use

The intended shape:

```bash
git checkout -b feat/rate-limiting
```

```
/pipeline add rate limiting to the login endpoint, 5 attempts per minute per IP, return 429 with Retry-After
```

Close the laptop. In the morning, read `.pipeline/review.md` first, then the diff. If
the verdict is SHIP, review it yourself and merge. If it's NEEDS WORK, the findings
are specific enough to act on directly.

Read the other three handoff files too, at least for the first several runs. Seeing
what the Planner needed in order to write a good spec is the fastest way to learn to
write better feature requests.

## Getting good results

**Be specific.** "Add rate limiting, 5 attempts per minute per IP, 429 with
`Retry-After`" produces a tight spec. "Add rate limiting" produces a page of open
questions.

**Start small.** One endpoint, one module, one bounded refactor. Trust the pipeline on
work you can verify at a glance before handing it something you can't.

**One feature per run.** The handoff files are single-slot. The command clears them at
the start of each run for exactly this reason.

**Parallel features need worktrees.** Two pipelines in one checkout will edit the same
files. Use `git worktree add` and run one per tree.

## Honest limitations

- **The Reviewer's read-only property is enforced by prompt, not by sandbox.** It has
  no `Edit` tool, but it does have `Bash` (it needs `git diff`) and `Write` (it needs
  to produce `review.md`). A determined model could route around the restriction. The
  instruction is explicit and models follow it in practice, but treat the guarantee as
  strong-convention, not airtight.
- **The Reviewer reads the same spec the Coder did.** A flawed spec faithfully
  implemented can still earn a SHIP. The Reviewer checks the code against the spec; it
  does not re-litigate whether the spec was right. You are the check on that.
- **Test quality is bounded by the repo's existing test setup.** The Tester matches
  what's already there and won't scaffold a framework from nothing.
- **It's not free.** Two Opus stages per feature. Small, well-scoped requests are
  cheaper and produce better specs anyway.

## Repo layout

```
.claude-plugin/
  plugin.json          plugin manifest
  marketplace.json     lets this repo be added as a marketplace directly
agents/
  pipeline-planner.md
  pipeline-coder.md
  pipeline-tester.md
  pipeline-reviewer.md
commands/
  pipeline.md          the orchestrator
```

To customize, edit the agent files and re-run `/plugin marketplace update`. The models
are set in each agent's frontmatter — swap `opus` for `sonnet` in the Planner and
Reviewer if you want a cheaper run, at a real cost to spec and review quality.

## Credit

```
   ╭──────────────────────────────────────────────────────────╮
   │  The idea behind this pipeline is Ray Fu's, not mine.    │
   ╰──────────────────────────────────────────────────────────╯
```

**Full credit for the design goes to [Ray Fu](https://www.skool.com/@ray-fu-5306).**

The four-role split, the `.pipeline/` handoff-file pattern, the model assignment
(Opus for planning and review, Sonnet for implementation and testing), and the
read-only-reviewer constraint all come from his write-up, *"How to Build a 4-Agent Dev
Team That Ships Features While You Sleep."* The insight that a reviewer which *can*
fix things stops honestly reporting them is his, and it's the load-bearing idea in the
whole design.

He walks members through the setup in his community — **[skool.com/raycfu](https://www.skool.com/raycfu)**.
If this pipeline is useful to you, that's where it came from.

This repo is an independent implementation of that design, packaged as an installable
Claude Code plugin. The agent prompts, the spec and review output formats, the
orchestrator's gating and preflight checks, and the `/pipeline` naming are rewritten
and extended here — but the architecture is Ray's.

## License

MIT — see [LICENSE](LICENSE).
