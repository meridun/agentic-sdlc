# Adoption guide

How to wire this template into a real repo. ~30 minutes for a first pass.

## 1. Copy the trees

- `prompts/sdlc/` → `prompts/sdlc/` in your repo (or anywhere your agent can read; the dispatcher just
  needs the path).
- `agents/sdlc-worker.md` → your harness's agent dir, with the matching frontmatter from that file:
  - Claude Code: `.claude/agents/<WORKER_AGENT>.md`
  - GitHub Copilot: `.github/agents/<WORKER_AGENT>.agent.md`

Name the agent something project-scoped (e.g. `acme-sdlc-worker`) and use that everywhere `<WORKER_AGENT>`
appears.

## 2. Create the labels

Run the `gh` script in [labels.md](labels.md). It creates the `stage:*`, `sdlc:*`, and `priority:*`
labels the prompts depend on.

## 3. Fill the placeholders

Grep for `<` across `prompts/` and `agents/` and replace every one. The full list:

| Placeholder | Meaning | Example |
|---|---|---|
| `<PROJECT>` | Project name (appears in every worker's opening line) | `Acme API` |
| `<REPO_PATH>` | Local working directory the scheduled agent runs in | `C:\work\acme` |
| `<WORKER_AGENT>` | The isolated worker agent's `subagent_type` name | `acme-sdlc-worker` |
| `<DEFAULT_BRANCH>` | Integration branch — PRs target it, branches cut from it | `dev` or `main` |
| `<PROD_BRANCH>` | Release branch — **off-limits** to all workers | `main` or `release` |
| `<WORKTREE_ROOT>` | Where issue-scoped worktrees live | `../acme-wt` |
| `<BUILD_CMD>` | Build/compile/launch the software for a real run | `npm run build` |
| `<TEST_CMD>` | Targeted test run (build stage) | `npm test -- <path>` |
| `<FULL_SUITE_CMD>` | Full test suite (verify stage) | `npm test` |
| `<LINT_CMD>` | Lint/format/type gate that must be clean before commit | `npm run lint` |
| `<SMOKE_CMD>` | Repeatable real-run / e2e / smoke spec (verify stage) | `npm run e2e` |
| `<LANG_CONVENTIONS>` | The lint/format/test bar in one line | `eslint clean, prettier applied, jest green` |
| `<INVARIANTS>` | Project rules that are ACs on **every** change — list them | `no breaking API changes; all endpoints authz-checked; no PII in logs` |
| `<DECISION_RECORD>` | Where decisions are logged (a doc section, a registry file) | `docs/Decisions.md` |
| `<DOCS_SINKS>` | Documentation targets ship fans out to | `README.md, docs/API.md` |
| `<TOKEN_TOOL>` | *(optional)* shell-output compactor to prefix commands with; delete the mentions if none | `rtk` |

`<INVARIANTS>` is the one that most repays effort — it is the shared acceptance criterion build
implements to, verify exercises, and audit reviews for. Be specific and concrete.

## 4. Copy the reference CLI (recommended)

Copy `tools/sdlc.mjs` into your repo (plain Node, no dependencies) and adapt the constants at the
top — `DEFAULT_BRANCH`, `PROD_BRANCH`, and the worktree naming function. Then wire the lane prompts
to it: wherever a prompt describes the claim lock, stage swap, or dispatcher gate ritual, have
workers run the CLI one-shot instead (`node tools/sdlc.mjs claim|advance|gate|lock|lanes|…`).

Why bother: `advance` validates every transition against the stage graph, which kills the
hand-typed label-typo class (`stage:verfy`) by construction, and it makes the gate, dispatcher
lock, and claim-verify race check deterministic — the agent supplies judgment (what to spawn, what
to write in comments), the CLI supplies the state math. The pure helpers are exported and the
gh/git executors injectable, so you can unit-test your adaptations without touching GitHub.

## 5. Choose your variant

- Small backlog / single tree → **serial** (see [AgenticSDLC.md](AgenticSDLC.md#concurrency-variants);
  simplify `dispatch.md` per the note there, and you can ignore the worktree steps).
- Throughput matters → keep the shipped **per-issue** model. Create the `sdlc:dispatch-lock` pinned
  issue once (`gh issue create --title sdlc:dispatch-lock --body "dispatcher mutex" --label sdlc:hold`,
  then pin it), and confirm your platform allows git worktrees.

Decide whether you want the optional `stage:design` lane (add the label + the shipped
`prompts/sdlc/design.md` worker) or fold design into intake (shipped default).

## 6. Write your conformance profile

Create `prompts/sdlc/PROFILE.md` from the skeleton in
[Composability.md](Composability.md#the-conformance-profile), declaring your bindings for the five
variation points and any known deviations. This is what keeps ad-hoc drift audits against the spec
cheap. A worked ADO example lives at [profiles/work-ado.example.md](profiles/work-ado.example.md).

## 7. Dry-run manually before scheduling

Paste `prompts/sdlc/README.md` + one stage file (start with `intake.md`) into an agent session pointed
at your repo. Feed it a real issue labeled `stage:intake`. Watch it CLAIM, WORK, and EMIT. The prompt
behaves identically whether a human or a scheduler fired it — so a clean manual pass means the
scheduled one will work. Walk one issue all the way through intake → ship this way before automating.

## 8. Schedule the dispatcher

Register a recurring task whose body is a **thin pointer** to `dispatch.md`, e.g.:

> Read `<REPO_PATH>/prompts/sdlc/dispatch.md` and execute one dispatch cycle for the `<PROJECT>`
> project. The file is the canonical prompt; follow it exactly.

Cadence: hourly is typical (the 2h reap threshold assumes ≤ hourly). Options:
- **Claude Code scheduled tasks** — the native path; the task calls the dispatcher subagent.
- **cron + headless agent** — `cron` → a headless agent invocation with the pointer as its prompt.
- **CI cron** (GitHub Actions `schedule:`) — a job that runs the agent CLI against the pointer.

**Enable it only once your queue depth justifies the spend** — each cycle costs tokens per non-empty
lane. Until then, run it manually on demand.

**Dispatcher sizing:** keep the dispatcher itself mid-tier (sonnet-class) even once its steps are
CLI-scripted — a dispatch failure is systemic (a whole cycle misroutes), not per-issue like a worker
failure. Consider dropping it to a small model (haiku-class) only after the worktree sweep and
conflict scan are scripted too and you've observed several clean cycles.

## 9. Watch the first few cycles

The dispatcher's digest (end of every run) is your dashboard: lock result, git maintenance, one line
per lane, queue depths, parked items, and **token cost per lane + cycle total**. That token line is the
trend to watch for cost regressions. Parked (`sdlc:needs-human`) items are your action queue.
