# The Agentic SDLC model

This is the *why* behind the prompts. Read it once; the prompts are self-contained after that.

## The idea

A backlog of GitHub issues, each carrying a `stage:` label, is a state machine. A **coding agent**
advances one issue one stage at a time. A **dispatcher**, on a schedule, spawns one **worker** per
stage. Workers are stateless and isolated — they carry nothing between runs; the **issue thread is the
entire shared memory** of the pipeline. That single constraint is what makes the system robust: any
worker can die mid-pass and the next scheduled run picks up exactly where the issue's labels and
comments say things stand.

```
 raw idea ─▶ intake ─▶ [queued] ─▶ build ─▶ verify ─▶ audit ─▶ ship ─▶ PR ─▶ (human merges) ─▶ closed
             triage    human       write     prove     review    docs +
             + route   throttle     code      it works  it's safe open PR
```

The **canonical spine** (see [Composability.md](Composability.md)) names nine stages:
`intake → [design] → queued → build → verify → audit → ready → shipping → complete`, with two
human gates (`queued` and `ready`). The shipped template implements the collapsed tail: `ship`
does docs + PR, the human merge **is** the `ready` gate, and `shipping → complete` collapse into
merge-and-close. Multi-repo forks make the tail explicit; both forms conform.

## The five invariants that make it safe

1. **One issue per pass, one outcome per pass.** A worker CLAIMs one issue, does the stage's work, and
   EMITs exactly one of `ADVANCE` / `BOUNCE` / `PARK` / `CONTINUE` — never silently. No worker ever
   processes two issues in a run. This bounds blast radius and makes every run auditable from the issue
   thread alone.

2. **Idempotency.** Schedulers fire on a clock, not on need. Every stage checks "is my artifact already
   present for this branch HEAD?" and no-ops if so. A re-run must never redo completed work or restart
   an in-progress branch — it *continues* it.

3. **Isolation — no delegation, no shared tree.** Workers have no agent-spawning tool (a spawned
   subagent runs detached and strands the item). They never work in the main checkout — each uses an
   issue-scoped git worktree, which doubles as a second lock (git refuses two checkouts of one branch).
   "Owner skill X" in a lane prompt means *apply X's checklist inline*, not *spawn X*.

4. **Stale-lock reaping, never live-lock stomping.** `sdlc:wip` is the lock. The dispatcher reaps a
   lock only when its claim comment is ≥2h old (two full hourly cycles — no legitimate pass runs that
   long). A fresh lock is a live worker and is left alone. Human-set labels (`sdlc:needs-human`,
   `sdlc:hold`) are never touched by any automation.

5. **Bounce to the lane that owns the failure; park for the human.** A red test bounces to build; a
   security defect bounces to build; an undecided question bounces to intake (or design); a risk
   tradeoff or a "the design itself is wrong" call PARKs to a human via `sdlc:needs-human`. Failures
   flow to accountability, not in a circle.

## The label protocol

- `stage:intake` · `stage:build` · `stage:verify` · `stage:audit` · `stage:ship` — the lane an issue
  is in. Exactly one at a time. Optionally `stage:design`.
- `stage:queued` — **workerless**. The human throttle between intake and build: the only gate a human
  must open by hand. This is what prevents a runaway pipeline from consuming build capacity on
  half-baked ideas.
- `sdlc:wip` — the per-issue lock. Machine-owned, volatile: workers set/clear it, the reaper may strip
  it. Paired with an `sdlc:claim <run-id> <lane>` comment that records ownership + timestamp.
- `sdlc:needs-human` — parked. A worker hit something only a human can decide. Automation never
  advances or reaps a parked item; it re-enters its lane when the human clears the label.
- `sdlc:hold` — human keep-off. No worker touches it. (Also used to mark the dispatcher-lock issue.)
- `priority:critical` › `priority:medium` › `priority:future` — CLAIM order within a lane, then FIFO by
  creation date.

Full `gh`-scriptable list: [labels.md](labels.md).

## Concurrency variants

The template ships the **per-issue** model. A simpler **serial** model exists — pick by backlog size.

### Per-issue (shipped default)
- Locking is per-issue via claim comments with run-ids + a claim-verify race check.
- The dispatcher holds a **singleton mutex** (the pinned `sdlc:dispatch-lock` issue) so two dispatchers
  never do maintenance at once, but lane workers themselves **run concurrently** — each in its own
  worktree.
- A fresh lock only removes *that one issue* from eligibility; it never aborts the cycle.
- Best when throughput matters and multiple issues are in flight across lanes.

### Serial (simpler alternative)
- No dispatcher mutex, no claim comments, no worktrees required.
- The wip gate is **global**: if *any* issue carries an `sdlc:wip` younger than 2h, the whole run
  **aborts** (a live worker exists somewhere). Older → reap and proceed.
- Lanes run **one at a time**, in pipeline order; workers operate in the main checkout with strict
  tree-hygiene (record and restore the entry branch, never stash human WIP).
- Best when the backlog is small, or when running in a single tree without worktree support.

To switch a shipped pipeline to serial: drop the dispatcher's Step -1 and the claim-comment steps,
replace Step 0's per-issue gate with the global abort-or-reap, and run the per-lane loop serially.

## Optional design lane

UI/UX-heavy work benefits from a `stage:design` lane between intake and queued: intake routes a
design-owed item to `stage:design`, a design worker storyboards/specs the change, and only then does it
reach `stage:queued`. CLI / library / backend projects skip the lane and fold design questions into
intake as **decision debates** — intake PARKs with the options framed in-issue, the human decides, and
intake records a `<DECISION_RECORD>` one-liner before routing onward. The worker prompt ships
(`prompts/sdlc/design.md`) but the default pipeline leaves the lane off; enable it (create the
`stage:design` label and dispatch the lane) if your work is user-facing and worth storyboarding.

## Why the issue thread is the only state

Everything a downstream stage needs, the upstream stage writes into the issue as a comment: build's
branch name and plan, verify's report and evidence, audit's findings. A worker reconstructs its entire
context from `gh issue view` + the branch. This is what lets the whole thing survive process death,
run headless on a cron, and be debugged by a human reading one issue top to bottom.
