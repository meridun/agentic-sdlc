# SDLC worker prompts

Executable artifacts (not reference docs). Each file is **one stage worker** for the agentic SDLC
pipeline. All share the universal worker loop below; lane files define only their WORK and EMIT
specifics.

Pipeline: `stage:intake` → `stage:queued` → `stage:build` → `stage:verify` → `stage:audit` →
`stage:ship` → *(PR merged, closed)*.

`stage:queued` is intentionally workerless — the human throttle between triage and build. Whether
you run a separate `stage:design` lane is a per-project choice; see the repo `docs/` — the default
pipeline folds design questions into intake as decision debates.

> **Placeholders.** Anything in `<ANGLE_BRACKETS>` is project-specific and must be filled in before
> use. See `docs/Adoption.md` for the full list. The core ones: `<PROJECT>` (name), `<REPO_PATH>`
> (local working dir), `<DEFAULT_BRANCH>` (integration branch, e.g. `dev`), `<PROD_BRANCH>`
> (release branch, off-limits to workers), `<WORKTREE_ROOT>` (e.g. `../<project>-wt`),
> `<TEST_CMD>` / `<FULL_SUITE_CMD>` / `<LINT_CMD>` / `<BUILD_CMD>` / `<SMOKE_CMD>`, `<INVARIANTS>`
> (project rules that are acceptance criteria on every change), and `<DECISION_RECORD>` (where
> decisions are logged). Optionally `<TOKEN_TOOL>` (a shell-output compactor to prefix commands
> with; omit if none).

## How to run

- **Scheduled (primary):** the `sdlc-dispatch` scheduled task runs the dispatcher prompt
  ([`dispatch.md`](dispatch.md)) on a cadence (hourly is typical): a dispatcher singleton gate, a
  per-issue wip gate (stale-lock reaping), git + worktree maintenance, then one `<WORKER_AGENT>`
  subagent per non-empty lane. Each worker reads this README plus its lane file and executes one
  pass; workers share no context — the issue thread is the only carried state. The dispatcher's own
  prompt is [`dispatch.md`](dispatch.md) — the canonical copy; the scheduled task is a thin pointer
  to it.
- **Manual:** paste this README plus a worker file's body into an agent session. Identical behavior
  — the prompt doesn't know what fired it. Mint your own run-id for the claim comment. Manual and
  scheduled runs coexist safely: claims deconflict per-issue.

## Universal worker loop (binding)

1. **CLAIM** — list open issues labeled `stage:<lane>` that are **NOT** labeled `sdlc:wip`,
   `sdlc:needs-human` (parked), or `sdlc:hold` (human keep-off). Pick the next: higher priority
   first (`priority:critical` › `priority:medium` › `priority:future`), then oldest by creation date
   (FIFO). If none → reply `<LANE>: idle` and stop. Then take the lock, in this order:
   1. Add `sdlc:wip` to the chosen issue.
   2. Post a claim comment: `sdlc:claim <run-id> <lane>` (run-id = the dispatcher-supplied id, or
      any unique id you mint for a manual run). The label is the visibility signal; the claim comment
      is the ownership record and tiebreaker.
   3. **Claim-verify:** re-fetch the issue's comments. If another `sdlc:claim` comment on this issue
      is newer than the last outcome EMIT and predates yours (or ties with a lexicographically lower
      run-id), you lost the race — leave the label and the winner's claim untouched, delete nothing,
      and go pick the next eligible item. Only the losing worker's own claim comment may be edited to
      note `superseded`.

   The lock is machine-owned and volatile; the dispatcher's reaper may strip it, and it checks the
   claim comment's run-id + timestamp before doing so.

   If your repo carries the reference CLI (`tools/sdlc.mjs`, see `docs/Adoption.md`), steps 1–3
   collapse into one deterministic command: `node tools/sdlc.mjs claim <issue> <run-id> <lane>
   --verify` adds the label, posts the claim comment, and runs the race check — a non-zero exit
   means you lost the race: walk away and pick the next eligible item.
2. **WORK** — per the lane file, with these constraints:
   - **Never delegate — do all work inline, yourself.** No subagents (in an async harness they run
     detached; the worker yields and nothing resumes it — the item strands under `sdlc:wip`), no
     background tasks, no wait loops. Where a lane prompt names an owner skill or checklist, that
     names the **approach you apply inline**, not a subagent to dispatch. (Scheduled runs enforce
     this structurally: workers are spawned as `<WORKER_AGENT>`, which has no delegation tool.)
   - **Idempotent — reconcile, don't re-execute.** If the stage's artifact already exists, treat as
     done; do not redo. Schedulers fire on a clock, not on need — a re-run must be a safe no-op. The
     same applies to items a human rewound to an earlier stage or reopened after close: investigate
     what already exists before doing any work. Evidence hierarchy: merged code / branch state / PR
     status › recorded reports for the current HEAD › issue comments › labels — cite what you relied
     on when you no-op. Presume an existing valid artifact good unless the human's rewind comment
     gives a reason to distrust it or your own check finds something significant; then redo exactly
     the invalidated part. Keep the check cheap — dig deeper only when evidence conflicts, and PARK
     if it stays ambiguous rather than burn the pass. For partial work, post a short reconciliation
     note (what's already done + evidence, what remains) before continuing, then do only the gap.
     If the item is conclusively shipped already, PARK with the evidence (PR#, commit, observed
     behavior) for a human to close — don't march it through the remaining lanes.
   - **Worktree isolation.** Never work in the main checkout — it may hold human WIP or another
     worker. For any lane that touches a branch, use the issue-scoped worktree
     `<WORKTREE_ROOT>/<issue#>`: create it if missing (`git worktree add <WORKTREE_ROOT>/<issue#>
     <branch>`, cutting the branch first if the lane owns branch creation), reuse it if present.
     Git's one-checkout-per-branch rule across worktrees is a second lock layer: if `worktree add`
     fails because the branch is checked out elsewhere, treat it as a lost claim race — release per
     CLAIM step 3 and move on. Do all git/build/test work inside the worktree; never stash, discard,
     or overwrite files in the main tree.
   - **Refresh from `<DEFAULT_BRANCH>` (staleness rule).** On entering the worktree:
     `git fetch origin`. If `git diff --name-only HEAD...origin/<DEFAULT_BRANCH>` (upstream side)
     intersects the paths this branch touches, `git merge origin/<DEFAULT_BRANCH>` (merge, never
     rebase — branches are pushed and handed between workers). No overlap → record "dev advanced, no
     path overlap" and do not merge, so existing verify/audit reports stay valid. **Conflict
     ownership:** build resolves merge conflicts; verify and audit never do — a conflicted merge
     there is a BOUNCE → `stage:build` naming the conflicting paths. Ship always merges (the PR must
     be mergeable) and may resolve docs-only conflicts itself; code conflicts BOUNCE to build.
   - **Tree hygiene.** Before any branch switch, record the entry branch
     (`git rev-parse --abbrev-ref HEAD`) and restore exactly it before EMIT — never `<PROD_BRANCH>`
     (release, off-limits) and never a guessed default. Never stash, discard, or overwrite
     uncommitted files you didn't create (human WIP); if they genuinely block the work, PARK.
3. **EMIT exactly one outcome** — ADVANCE, BOUNCE, or PARK (build also defines CONTINUE) — never
   silent. **Every outcome removes `sdlc:wip`** on the way out. Leave the worktree in place
   (dispatcher maintenance prunes worktrees for merged/dead branches).
4. **STOP** — reply the lane's one-line result. One item per pass; never pick up a second.

## Project invariants (bind in every lane)

Fill these in for your project — they are acceptance criteria on **every** change, checked at build,
proven at verify, and audited at audit:

- **Git flow is `{feature} → <DEFAULT_BRANCH> → <PROD_BRANCH>`.** Feature branches
  `<type>/<issue#>-<slug>` cut from `<DEFAULT_BRANCH>` (the default/integration branch); PRs target
  `<DEFAULT_BRANCH>`; the human merges. `<PROD_BRANCH>` is the stable/release branch — it moves only
  by human-initiated PR. **No worker ever branches from, checks out, commits to, or targets
  `<PROD_BRANCH>`.**
- **`<INVARIANTS>`** — the project-specific rules that must hold on every change (e.g. exit-code
  parity, backward-compatible schema, no secrets in fixtures, multiplayer-safe state). List them
  explicitly so build implements to them, verify exercises them, and audit reviews for them.
- **`<LANG_CONVENTIONS>`** — the lint/format/test bar (e.g. `<LINT_CMD>` clean, `<FULL_SUITE_CMD>`
  green, formatter applied) that gates every commit.
- **Decisions live in `<DECISION_RECORD>`, not in doc prose or build-branch cargo** — a decision is
  a one-liner linking to the issue that holds the debate. Workers never write ADR-style history into
  docs.

## Files

| File | Stage | Notes |
|---|---|---|
| [`dispatch.md`](dispatch.md) | *(dispatcher — runs every lane)* | scheduled task; git/worktree maintenance + per-lane fan-out |
| [`intake.md`](intake.md) | `stage:intake` → `stage:queued` | triage, dedup, decision debates + merge sweep |
| [`design.md`](design.md) | `stage:design` → `stage:queued` | *(optional lane)* settle approach/UX before build; omit if folding design into intake |
| [`build.md`](build.md) | `stage:build` → `stage:verify` | plan comment → implement + targeted tests |
| [`verify.md`](verify.md) | `stage:verify` → `stage:audit` | full suite + real-run smoke |
| [`audit.md`](audit.md) | `stage:audit` → `stage:ship` | read-only security/invariant review of the diff |
| [`ship.md`](ship.md) | `stage:ship` → *(closed on merge)* | docs fan-out + open PR |

`intake` additionally runs a **merge sweep** every pass (its step 0): post-merge cascade-unblock for
issues closed by merged PRs, since ship ends at "PR open" and the merge itself fires no worker.
