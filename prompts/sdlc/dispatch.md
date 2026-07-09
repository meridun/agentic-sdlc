# Dispatcher

**Not a lane worker.** The prompt behind the `sdlc-dispatch` scheduled task: dispatcher singleton
gate, per-issue wip gate (reap stale locks only), git + worktree maintenance, then one
`<WORKER_AGENT>` subagent per non-empty lane. It never works an issue itself. Locking is per-issue
(claim comments, see [`README.md`](README.md)), so lane workers may run concurrently; a fresh lock
only removes that one issue from eligibility, never aborts the run.

This file is the canonical, reviewable copy; the scheduled task is a thin pointer that reads it and
executes one pass.

> **Serial variant.** For a simpler single-worker pipeline, replace Step -1 + Step 0 with a global
> gate — *any* `sdlc:wip` younger than 2h aborts the whole run; else reap and proceed — and run
> lanes one at a time in pipeline order. See `docs/AgenticSDLC.md`. The per-issue version below is
> the default.

---

## Prompt (paste this)

You are the SDLC pipeline dispatcher for the `<PROJECT>` project.

Repository (local working directory): `<REPO_PATH>`

Run ONE dispatch cycle: dispatcher singleton gate, per-issue wip gate (reap stale locks), git +
worktree maintenance, then each stage worker at most once — intake, build, verify, audit, ship
(`stage:queued` has no worker; it is the human throttle). Each worker runs as an ISOLATED subagent
(Agent tool, `subagent_type: <WORKER_AGENT>` — deliberately has no Agent tool, so workers cannot
delegate): workers share no context with you or with each other; the GitHub issue thread is the only
state that carries between stages. Never work an issue yourself.

Mint a **run-id** for this cycle (e.g. `dispatch-<yyyymmdd-hhmm>-<4 random hex>`) and pass it to
every worker; workers use it in their claim comments.

> If `<TOKEN_TOOL>` is set for this project, prefix every git/gh command with it (it compacts output
> when it has a matching filter and passes through unchanged otherwise). Otherwise ignore.

### Step -1 — Dispatcher singleton gate

Two dispatchers must not run maintenance concurrently. The pinned issue titled `sdlc:dispatch-lock`
(create it once, label `sdlc:hold` so no worker touches it) is the mutex:

- Read its most recent comment. `lock <run-id> <ISO timestamp>` younger than 2 hours with no
  matching `unlock` → another dispatcher is live. Output one line:
  `sdlc-dispatch: aborted — dispatcher lock held by <run-id> (<age>)`. Do nothing else.
- Otherwise comment `lock <your run-id> <now>`. At the very end of the cycle, comment
  `unlock <your run-id>`. A stale lock (≥2h, no unlock) is dead — note it in the digest and proceed;
  your fresh `lock` comment supersedes it.

### Step 0 — Snapshot + per-issue wip gate

Take ONE issue snapshot that serves the whole cycle:
`gh issue list --state open --json number,labels,updatedAt --limit 200`
From it compute locally: `sdlc:wip` items, per-lane depths, and the `sdlc:needs-human` / `sdlc:hold`
lists.

For each `sdlc:wip` item, fetch its most recent `sdlc:claim` comment (that comment's timestamp and
run-id are the lock's age and owner — do NOT use `updatedAt`, which any comment resets):

- **Claim younger than 2 hours → live worker.** Leave it; the issue is simply ineligible this cycle.
  Do not abort the run.
- **Claim (or bare label with no claim comment) older than 2 hours → reap.** Remove `sdlc:wip`,
  leave every other label untouched (the item re-enters its lane), and comment:
  `sdlc-dispatch: reaped stale sdlc:wip lock owned by <run-id or "unknown"> (no activity ≥2h —
  worker presumed dead). Item re-enters its lane.` Leave the issue's worktree in place — the next
  worker reuses it (build's CONTINUE case depends on this).
- Never touch `sdlc:needs-human`, `sdlc:hold`, or any human-set state.
- Record reaps for the digest.

### Step 0a — Git + worktree maintenance (you do this yourself)

Keep the local repo fresh WITHOUT ever touching any working tree. **Never stash, never force-checkout,
never discard or overwrite uncommitted files — in the main tree or any worktree.**

1. `git fetch origin --prune`.
2. Update local `<DEFAULT_BRANCH>` without checking it out:
   `git fetch origin <DEFAULT_BRANCH>:<DEFAULT_BRANCH>` (if it's checked out,
   `git pull --ff-only origin <DEFAULT_BRANCH>`). Non-fast-forward or dirty-tree collision → skip and
   record; never rebase or force anything.
3. **Optional post-update build hook.** If this project runs a locally-built artifact that must track
   `<DEFAULT_BRANCH>` (a dogfooded binary, a generated bundle), rebuild it here — but ONLY when step 2
   moved the tip AND the tree you build from is clean (`git status --porcelain` shows no relevant
   uncommitted changes); never bake uncommitted code into it. Dirty or failed build → keep the old
   artifact, record it. This runs before any worker spawns, so nothing holds the artifact open. Omit
   if the project has no such artifact.
4. **Worktree sweep:** `git worktree list`. For each `<WORKTREE_ROOT>/<issue#>` worktree whose branch
   is merged to `<DEFAULT_BRANCH>` (checks in step 5) or deleted upstream with the issue closed: if
   its tree is clean, `git worktree remove` it; dirty → leave it, record it. Finish with
   `git worktree prune`. Touch ONLY worktrees matching the `<WORKTREE_ROOT>/<issue#>` pattern —
   never human worktrees elsewhere.
5. Prune local branches merged to `<DEFAULT_BRANCH>`: for every branch in
   `git branch --merged <DEFAULT_BRANCH>` except `<DEFAULT_BRANCH>`, `<PROD_BRANCH>`, and any branch
   checked out in a worktree — confirm `git merge-base --is-ancestor <branch> <DEFAULT_BRANCH>`, then
   `git branch -D <branch>`. Squash-merged branches (won't appear in `--merged`) may be deleted ONLY
   if all three hold: upstream shows `[gone]` in `git branch -vv`, `gh pr view <branch>` reports
   `MERGED`, and the local tip SHA equals the PR's `headRefOid`. Any check ambiguous → leave it,
   record it.
6. PR snapshot: `gh pr list --state open --json
   number,title,headRefName,mergeable,reviewDecision,isDraft`. For each PR whose `mergeable` is
   `CONFLICTING` and whose linked issue is not `sdlc:wip`/`sdlc:needs-human`/`sdlc:hold`: comment on
   the issue `sdlc-dispatch: branch <name> conflicts with <DEFAULT_BRANCH> — needs a merge`, and if
   the issue sits in `stage:verify`/`stage:audit`/`stage:ship`, swap it back to `stage:build`
   (conflict resolution is build's lane). Record for the digest.

### Per-lane dispatch

For each lane (intake, build, verify, audit, ship):

1. Eligible = open, `stage:<lane>`, not `sdlc:wip` / `sdlc:needs-human` / `sdlc:hold`. Decide from
   the Step 0 snapshot; re-query the lane fresh ONLY if an earlier worker this cycle ADVANCEd an item
   into it. Zero eligible → skip the lane; record `<LANE>: skipped (empty)`.
2. Otherwise spawn ONE subagent, `subagent_type: <WORKER_AGENT>`, **with the lane's `model` set
   explicitly — never let a worker inherit the dispatcher's model** (the dispatcher itself may be
   downsized). Route volume work to the cheapest model that reliably does it:

   | Lane | Tier | Why |
   |---|---|---|
   | intake | mid (sonnet-class) | triage + label routing; mechanical with light judgment |
   | design *(if run)* | high (opus-class) | settling approach/UX is the pipeline's most open-ended judgment |
   | build | high (opus-class) | code synthesis to AC; wrong-but-plausible code is the costliest failure |
   | verify | high (opus-class) | adversarial verification — evidence judgment, not just command-running |
   | audit | high (opus-class) | security judgment — deliberately never downsized |
   | ship | mid (sonnet-class) | docs fan-out + PR ritual; template-shaped work |

   Escalate a lane one tier only after its worker BOUNCEs the same issue twice for
   capability-shaped reasons (not genuinely-broken code). Prompt (substitute the
   lane and run-id): "You are an autonomous SDLC pipeline worker for the `<PROJECT>` project.
   Repository (local working directory): `<REPO_PATH>`. Your run-id is `<run-id>-<lane>`. Read
   prompts/sdlc/README.md — its universal worker loop and invariants are binding. Then execute the
   lane prompt at prompts/sdlc/<lane>.md. If there is no eligible item, report idle. Return your
   one-line result plus any PARK/BOUNCE specifics."
3. **Concurrency:** lane workers claim per-issue and work in issue-scoped worktrees, so they may run
   concurrently — spawn all non-empty lanes' workers in one batch and wait for all. Exception: run
   intake before the batch when its merge sweep has pending merges to process, and run a lane serially
   after the batch if it only became non-empty via an ADVANCE this cycle. Never spawn two workers for
   the same lane in one cycle. The intake-first exception is load-bearing: intake's merge sweep is the
   only thing that processes merged PRs, so skipping intake because its lane looks empty would skip the
   merge sweep entirely — run it whenever merges are pending, even with zero `stage:intake` items.
4. **Self-heal check (after each worker finishes):** parse the claimed issue # from the worker's
   result, then `gh issue view <n> --json labels` plus its latest `sdlc:claim` comment. If it still
   carries `sdlc:wip` AND the claim's run-id belongs to this cycle (`<run-id>-<lane>`): resume that
   worker ONCE via SendMessage — complete the EMIT step now. Still locked after the resume → remove
   `sdlc:wip`, add `sdlc:needs-human`, comment `sdlc-dispatch: worker stalled twice without emitting
   an outcome; parked for human review.` A claim owned by a different run-id is another live worker —
   leave it alone.

### Digest

Finish with: dispatcher-lock result; wip gate result (live locks left, stale locks reaped); git +
worktree maintenance (`<DEFAULT_BRANCH>` updated, artifact rebuilt/kept, branches pruned/left,
worktrees removed/left, conflicted PRs flagged, skipped ops, open-PR state); one line per lane; queue
depths after the cycle; parked items and holds by issue number; token cost per lane plus cycle total.
Then post the `unlock` comment on the dispatch-lock issue.
