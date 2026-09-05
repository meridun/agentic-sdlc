# agentic-sdlc

A portable, project-agnostic **agentic SDLC pipeline**: a set of prompts and agent definitions
that let a coding agent (Claude Code, or any harness with subagents + an issue tracker) run
your backlog through a staged software-delivery pipeline — intake → design → queued → build →
verify → audit → ship (`design` produces the implementation plan a human reviews at `queued`, the
workerless human throttle) — one issue per stage, unattended, on a schedule.

It is a **template**, not a framework. There is no runtime to install. You copy the `sdlc/` tree
and one agent file into a repo, pick a **binding** (GitHub issues ships as the default; two Azure
DevOps shapes are included), fill in **one profile file**, register one scheduled task, and the
pipeline runs itself against your tracker. The tree is layered — **core** prompts you never edit,
a **binding** per tracker substrate you pick, a **profile** you fill — so upstream sync is an
overwrite, not a merge ([`sdlc/README.md`](sdlc/README.md)).

Distilled from two production pipelines (a multiplayer web game and a Go CLI tool). The hard-won
invariants — isolation, idempotency, stale-lock reaping, read-only audit, no-delegation workers —
are baked in; only the project-specific parts are placeholders.

---

## The model in one screen

```
 intake → design → queued → build → verify → audit → ready → shipping → complete
          spec     HUMAN                             HUMAN   └──── collapses to ────┘
          always   GATE 1                            GATE 2     the human PR merge
```

That is the **canonical nine-stage spine** ([docs/Composability.md](docs/Composability.md)). Read it
first so the shipped pipeline never surprises you — everything below is a *simplification* of it, not
a different model:

- **`design` is a standard stage.** Its **spec track** always runs: every item gets a reviewed
  implementation plan in the issue body (spec-lite for small items), which is exactly what the
  human approves or vetoes at the `queued` gate. Only its **UX/storyboard track** is optional —
  bind it (`<DESIGN_ARTIFACTS>`) for UI-facing work.
- **`ready → shipping → complete` collapse.** In a single-repo pipeline the human PR merge *is* the
  `ready` gate, and `shipping → complete` fold into merge-and-close. Multi-repo forks make the tail
  explicit; both forms conform.

So the **template you actually copy** ships six stage workers — `intake`, `design`, `build`,
`verify`, `audit`, `ship` — plus the workerless `queued` throttle. Only the tail collapses:

```
  ┌─────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
  │ intake  │──▶│ design │──▶│ queued │──▶│ build  │──▶│ verify │──▶│ audit  │──▶│  ship  │──▶ (PR merged)
  └─────────┘   └────────┘   └────────┘   └────────┘   └────────┘   └────────┘   └────────┘
   triage,       implementation human      execute the   full suite   security /   docs fan-out,
   dedup,        plan (+ UX     throttle   plan,         + real run   invariant    open PR
   route         when bound)    (workerless) targeted test            review
```

- **Each stage is one prompt** in [`sdlc/lanes/`](sdlc/lanes/). A worker runs exactly one pass
  of one stage against one issue, then stops.
- **The issue's evidence record is the only shared state.** Workers share no context with each
  other or the dispatcher — everything a downstream stage needs, the upstream stage writes into
  the issue.
- **A stage is a marker.** `stage:intake` … `stage:ship` — a label on GitHub, a tag on ADO; the
  binding picks the substrate, the name is canonical. Moving an issue forward = swapping its
  `stage:` marker. `stage:queued` has no worker — it is the human throttle between design and
  build, where a human approves (or vetoes) the implementation plan.
- **A [dispatcher](sdlc/dispatch.md)** runs on a schedule (hourly is typical): it does git
  maintenance, then spawns one isolated worker subagent per non-empty lane.
- **Every worker emits exactly one outcome** — `ADVANCE`, `BOUNCE`, `PARK`, or `CONTINUE` — and
  never silently. A bounce sends the issue back to the lane that owns the failure; a park hands it to
  a human via `sdlc:needs-human`.

The full narrative — why each rule exists, the failure modes it prevents — is in
[docs/AgenticSDLC.md](docs/AgenticSDLC.md).

---

## What's in here

| Path | What it is |
|---|---|
| [`sdlc/`](sdlc/README.md) | **The tree you copy.** Core + bindings + profile template + tracker-neutral tools. |
| [`sdlc/README.md`](sdlc/README.md) | The **universal worker loop** — CLAIM → WORK → EMIT → STOP — binding on every lane, plus the `<KEY>` / `` `op` `` resolution rules. Read this first. |
| [`sdlc/dispatch.md`](sdlc/dispatch.md) | The **dispatcher** prompt behind the scheduled task. Concurrent-safe (no singleton): wip reaping (verify-before-write), machine-locked git/worktree maintenance, per-lane fan-out. |
| [`sdlc/lanes/{intake,design,build,verify,audit,ship}.md`](sdlc/lanes/) | The six **stage workers**. Each defines only its own WORK and EMIT specifics; tracker operations are named abstractly. |
| [`sdlc/bindings/`](sdlc/bindings/README.md) | The **operation contract** every binding fills, and the bindings: [`gh-issue`](sdlc/bindings/gh-issue/BINDING.md) (GitHub issues + PRs, ships the reference CLI and the label taxonomy), [`ado-feature`](sdlc/bindings/ado-feature/BINDING.md) (Azure DevOps Feature / PBI / Task, multi-repo), [`ado-pbi`](sdlc/bindings/ado-pbi/BINDING.md) (Azure DevOps PBI / Task, single repo — proposed). |
| [`sdlc/bindings/gh-issue/sdlc.mjs`](sdlc/bindings/gh-issue/sdlc.mjs) | The **reference CLI** (plain Node, zero deps) — deterministic claim/advance/gate/lock state math so agents supply judgment, not label typing. |
| [`sdlc/PROFILE.md`](sdlc/PROFILE.md) | The **profile template** — the one file adoption fills: binding, every `<KEY>`, variation points, deviations. |
| [`sdlc/tools/check-lint-baseline.mjs`](sdlc/tools/check-lint-baseline.mjs) | The tracker-neutral **lint ratchet** for repos with a lint backlog. |
| [`agents/sdlc-worker.md`](agents/sdlc-worker.md) | The **isolated worker agent** definition — deliberately has no delegation tool. Ships in Claude Code (`.claude/agents/`) and GitHub Copilot (`.github/agents/`) frontmatter variants. |
| [`skills/documentation-tiers/SKILL.md`](skills/documentation-tiers/SKILL.md) | The **docs-tier discipline** the ship stage's docs fan-out routes to — hub-and-spoke L1/L2/L3 tiering, naming, sizing, thematic placement. Optional; copy into your harness's skill dir. |
| [`docs/AgenticSDLC.md`](docs/AgenticSDLC.md) | The model, the invariants, and the two concurrency variants (serial vs. per-issue). |
| [`docs/Adoption.md`](docs/Adoption.md) | Step-by-step: copy the tree, pick the binding, fill the profile, schedule the dispatcher. |
| [`docs/Composability.md`](docs/Composability.md) | **One spec, many forks** — the 9-stage canonical spine, the five variation points (tracker, topology, modules, dispatcher, quality bars), and the per-fork conformance profile. |
| [`docs/profiles/`](docs/profiles/) | **Worked profiles** for each binding: [`gh-issue`](docs/profiles/gh-issue.example.md), [`ado-feature`](docs/profiles/ado-feature.example.md), [`ado-pbi`](docs/profiles/ado-pbi.example.md). |

---

## Quick start

1. **Copy** `sdlc/` into your repo as `sdlc/`, and `agents/sdlc-worker.md` under
   `.claude/agents/` and/or `.github/agents/`.
2. **Pick the binding** — `gh-issue` for GitHub (then create the labels: run the script in
   [`sdlc/bindings/gh-issue/labels.md`](sdlc/bindings/gh-issue/labels.md)); `ado-feature` or
   `ado-pbi` for Azure DevOps.
3. **Fill the profile** — [`sdlc/PROFILE.md`](sdlc/PROFILE.md): the binding, every `<KEY>`
   (`PROJECT`, `REPO_PATH`, `DEFAULT_BRANCH`, `TEST_CMD`, `INVARIANTS`, …), the variation points,
   and any deviations. The prompts are never edited; [docs/Adoption.md](docs/Adoption.md) explains
   every key.
4. **Dry-run manually** — paste `sdlc/README.md`, the profile, the binding, and one lane file into
   an agent session and watch it process a single issue. The prompt doesn't know what fired it;
   manual and scheduled runs are identical.
5. **Schedule the dispatcher** — register a recurring task whose body is a thin pointer to
   `sdlc/dispatch.md`. Enable it once your queue depth justifies the spend.

---

## Two concurrency variants

The template ships the **per-issue** variant (workers claim individual issues with run-id comments
and run in isolated worktrees, so lanes run **concurrently**). A simpler **serial** variant (one
worker at a time; a global lock aborts the run if any worker is live) is documented alongside it in
[docs/AgenticSDLC.md](docs/AgenticSDLC.md#concurrency-variants). Start serial if your backlog is
small; move to per-issue when throughput matters.

## The design stage: spec always, UX when bound

Every triaged item passes through `stage:design` for an **implementation plan** written into the
issue body — the artifact the human reviews at the `queued` gate (spec-lite for small items, so a
bug fix costs a few lines, not a ceremony). UI/UX-heavy projects additionally bind the **UX
track** (`<DESIGN_ARTIFACTS>`): competing storyboards, a human A/B/C pick parked in-phase, then
the spec. CLI/library/backend projects leave it unbound and keep product/scope questions at intake
as decision debates. [docs/AgenticSDLC.md](docs/AgenticSDLC.md#the-design-stage--standard-with-two-tracks)
shows both tracks; the worker prompt is
[sdlc/lanes/design.md](sdlc/lanes/design.md).

---

## Requirements

- A coding agent that can spawn **isolated subagents** with a restricted toolset (this template
  targets Claude Code; the prompts are harness-agnostic prose and port to others).
- A tracker with a binding: **GitHub issues** + the `gh` CLI (`repo` scope, `gh` ≥ 2.86 for native
  dependencies) is the default; **Azure DevOps** via the `ado-feature` / `ado-pbi` bindings. A new
  tracker is a new `sdlc/bindings/<name>/BINDING.md` against the contract in
  [`sdlc/bindings/README.md`](sdlc/bindings/README.md).
- A scheduler for the dispatcher (Claude Code scheduled tasks, cron + headless agent, CI cron, etc.).

## Contributing to this repo

This repo follows its own template: **`feature → dev → master`**. Cut branches from `dev`, target
PRs at `dev`, and let `master` move only by a human-initiated `dev → master` promotion PR (merged
back into `dev` afterwards). See [CLAUDE.md](CLAUDE.md).

## License

[MIT](LICENSE).
