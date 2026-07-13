# agentic-sdlc

A portable, project-agnostic **agentic SDLC pipeline**: a set of prompts and agent definitions
that let a coding agent (Claude Code, or any harness with subagents + a GitHub issue tracker) run
your backlog through a staged software-delivery pipeline — intake → build → verify → audit → ship —
one issue per stage, unattended, on a schedule.

It is a **template**, not a framework. There is no runtime to install. You copy the `prompts/` and
`agents/` trees into a repo, fill in the `<PLACEHOLDERS>`, register one scheduled task, and the
pipeline runs itself against your GitHub issues.

Distilled from two production pipelines (a multiplayer web game and a Go CLI tool). The hard-won
invariants — isolation, idempotency, stale-lock reaping, read-only audit, no-delegation workers —
are baked in; only the project-specific parts are placeholders.

---

## The model in one screen

```
  ┌─────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
  │ intake  │──▶│ queued │──▶│ build  │──▶│ verify │──▶│ audit  │──▶│  ship  │──▶ (PR merged)
  └─────────┘   └────────┘   └────────┘   └────────┘   └────────┘   └────────┘
   triage,      human         cut branch,   full suite   security /   docs fan-out,
   dedup,       throttle      implement,    + real run   invariant    open PR
   route        (workerless)  targeted test              review
```

- **Each stage is one prompt** in [`prompts/sdlc/`](prompts/sdlc/). A worker runs exactly one pass
  of one stage against one issue, then stops.
- **The GitHub issue thread is the only shared state.** Workers share no context with each other or
  the dispatcher — everything a downstream stage needs, the upstream stage writes into the issue.
- **A stage is a label.** `stage:intake` … `stage:ship`. Moving an issue forward = swapping its
  `stage:` label. `stage:queued` has no worker — it is the human throttle between triage and build.
- **A [dispatcher](prompts/sdlc/dispatch.md)** runs on a schedule (hourly is typical): it does git
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
| [`prompts/sdlc/README.md`](prompts/sdlc/README.md) | The **universal worker loop** — CLAIM → WORK → EMIT → STOP — binding on every lane. Read this first. |
| [`prompts/sdlc/dispatch.md`](prompts/sdlc/dispatch.md) | The **dispatcher** prompt behind the scheduled task. Singleton gate, wip reaping, git/worktree maintenance, per-lane fan-out. |
| [`prompts/sdlc/{intake,build,verify,audit,ship}.md`](prompts/sdlc/) | The five **stage workers**. Each defines only its own WORK and EMIT specifics. |
| [`agents/sdlc-worker.md`](agents/sdlc-worker.md) | The **isolated worker agent** definition — deliberately has no delegation tool. Ships in Claude Code (`.claude/agents/`) and GitHub Copilot (`.github/agents/`) frontmatter variants. |
| [`skills/documentation-tiers/SKILL.md`](skills/documentation-tiers/SKILL.md) | The **docs-tier discipline** the ship stage's docs fan-out routes to — hub-and-spoke L1/L2/L3 tiering, naming, sizing, thematic placement. Optional; copy into your harness's skill dir. |
| [`docs/AgenticSDLC.md`](docs/AgenticSDLC.md) | The model, the invariants, and the two concurrency variants (serial vs. per-issue). |
| [`docs/Adoption.md`](docs/Adoption.md) | Step-by-step: labels to create, placeholders to fill, the scheduled task to register. |
| [`docs/labels.md`](docs/labels.md) | The `stage:*` / `sdlc:*` / `priority:*` label taxonomy, with a `gh` script to create them. |
| [`docs/Composability.md`](docs/Composability.md) | **One spec, many forks** — the 9-stage canonical spine, the five variation points (tracker, topology, modules, dispatcher, quality bars), and the per-fork conformance profile. |

---

## Quick start

1. **Copy** `prompts/` and `agents/` into your repo (agents go under `.claude/agents/` and/or
   `.github/agents/`).
2. **Create the labels** — run the script in [docs/labels.md](docs/labels.md).
3. **Fill the placeholders** — grep for `<` across `prompts/`; [docs/Adoption.md](docs/Adoption.md)
   lists every one and what it means (`<PROJECT>`, `<REPO_PATH>`, `<DEFAULT_BRANCH>`, `<TEST_CMD>`,
   `<INVARIANTS>`, …).
4. **Dry-run manually** — paste `prompts/sdlc/README.md` + one stage file into an agent session and
   watch it process a single issue. The prompt doesn't know what fired it; manual and scheduled runs
   are identical.
5. **Schedule the dispatcher** — register a recurring task whose body is a thin pointer to
   `prompts/sdlc/dispatch.md`. Enable it once your queue depth justifies the spend.

---

## Two concurrency variants

The template ships the **per-issue** variant (workers claim individual issues with run-id comments
and run in isolated worktrees, so lanes run **concurrently**). A simpler **serial** variant (one
worker at a time; a global lock aborts the run if any worker is live) is documented alongside it in
[docs/AgenticSDLC.md](docs/AgenticSDLC.md#concurrency-variants). Start serial if your backlog is
small; move to per-issue when throughput matters.

## Optional: a design lane

UI/UX-heavy projects benefit from a `stage:design` lane between intake and queued (storyboard the
change before building it). CLI/library/backend projects fold that into intake as a decision debate.
[docs/AgenticSDLC.md](docs/AgenticSDLC.md#optional-design-lane) shows both; the shipped pipeline omits
the lane — add it if your work is player-/user-facing.

---

## Requirements

- A coding agent that can spawn **isolated subagents** with a restricted toolset (this template
  targets Claude Code; the prompts are harness-agnostic prose and port to others).
- **GitHub issues** + the `gh` CLI, authenticated with `repo` scope.
- A scheduler for the dispatcher (Claude Code scheduled tasks, cron + headless agent, CI cron, etc.).

## License

[MIT](LICENSE).
