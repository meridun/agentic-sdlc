# SDLC conformance profile: single-repo GitHub — worked example

The minimal shape most forks start from: one GitHub repository, issues as the tracker, the
reference CLI installed, the collapsed spine tail. Distilled from a live single-repo adoption;
the project name is genericized to `acme`. Bindings per [Composability.md](../Composability.md).

- **Spine:** `intake → design → queued → build → verify → audit → ship` with the **collapsed
  tail** — ship opens the PR, the human merge **is** the `ready` gate, and `shipping → complete`
  collapse into merge-and-close. Design is standard (spec track always; UX track bound — see
  VP3); the only bypass is intake's already-built → `stage:verify` floor.
- **VP1 tracker: GitHub issues**, standard label taxonomy ([Labels.md](../Labels.md)).
  | Abstract operation | Binding |
  |---|---|
  | stage marker | exactly one `stage:*` label (`intake`, `design`, `queued`, `build`, `verify`, `audit`, `ship`) |
  | routing marker | n/a — single repo (VP2) |
  | claim lock | `sdlc:wip` label + claim comment `sdlc:claim <run-id> <lane>` (label = visibility; comment = ownership record and race tiebreaker) |
  | park to human | `sdlc:needs-human` + comment stating what is blocked and why |
  | human keep-off | `sdlc:hold` |
  | evidence record | issue **body sections** for durable artifacts (`## Requirements` / `## Acceptance criteria` from intake, `## Design` / `## Implementation plan` from design); comments for protocol traffic (claims, emits, routing summaries, verify report, audit findings); PR link on ship |
  | hierarchy & ordering | flat — `priority:critical` › `priority:medium` › `priority:future`, then FIFO by creation date |
  | label mutation safety | **all claim/emit/gate/lock label math via the reference CLI** (`tools/sdlc.mjs`), never hand-typed |
- **VP2 topology:** single repo — each issue is simultaneously the Feature and its only child. No
  hierarchy links, no routing tags, no contract-ready handshake.
- **VP3 modules:** design UX track **bound** (the product is UI-facing): competing storyboards per
  the project's mockup conventions, human A/B/C pick parked in-phase, decision graduated to the
  decision record. PSI lane **off**.
- **VP4 dispatcher:** scheduled agent task (hourly) → `dispatch.md` → one worker subagent per
  non-empty lane (worker agent has no delegation tool). No dispatcher singleton — concurrent runs
  deconflict via per-issue claims, idempotent verify-before-write GitHub writes, and the
  per-machine maintenance lock (`.git/sdlc-maint.lock`, atomic mkdir, 30-min stale reap);
  issue-scoped worktrees at `../acme-wt/<issue#>`. Workers end replies with the fenced JSON result
  block (README STOP contract) the dispatcher consumes.
- **VP5 quality bars:** targeted `npm run test:file <path>` · full suite `npm test` · smoke/e2e
  `npm run e2e` · lint gate `npm run lint:baseline` (ratchet — no new ESLint errors vs
  `tools/lint-baseline.json`, touched files clean; `npm run lint` is the interactive/human
  command) · invariants: multi-actor safety (never assume single-user
  state), defensive at boundaries (guard `undefined`, especially DB results), data access only in
  the repository layer · known env limits: none declared · docs sinks: `README.md` + the `docs/`
  tree · dep audit `npm audit --json` (intake sweep + audit lockfile check) · migrations
  `db/migrations/` with `dbmate down` / `dbmate up` and schema dump `db/schema.sql` (verify
  migration checks + audit diff-shape check).
- **Deterministic core:** `tools/sdlc.mjs` — owns claim/emit/advance transition validation, the
  wip gate, the maintenance lock, the native-dependency eligibility gate + derived readiness labels (`deps`), the close-sweep work-list/ack, and the cycle-prep report.
- **Known deviations from spec:** none — this is the template's own default shape, declared so
  drift audits have a baseline to diff against.
