# SDLC profile: work — worked example (`ado-feature`)

A read-only-consumer fork: the framework repo cannot be installed from; agents read the spec, the
binding, and this profile and maintain the local implementation by hand (Copilot agent files,
`sdlc.ps1`, a local `SDLC-PIPELINE.md`). The substrate contract — hierarchy, tag budget, rev-CAS
locking, status block, stage Tasks, PBI state machine — is the binding
([`ado-feature/BINDING.md`](../../sdlc/bindings/ado-feature/BINDING.md)); this profile carries
only what is project-specific. Bindings per [Composability.md](../Composability.md).

Spec version: `v2.0.0`

## Keys

| Key | Value |
|---|---|
| `BINDING` | `ado-feature` |
| `SDLC_CLI` | `pwsh ./sdlc.ps1` (local implementation of the binding's operation table) |
| `SPEC_VERSION` | `v2.0.0` |
| `PROJECT` | `work` |
| `REPO_PATH` | per repo: `core-api`, `webapp`, `utility` checkouts |
| `WORKER_AGENT` | `work-sdlc-worker` (`.github/agents/work-sdlc-worker.agent.md`) |
| `DEFAULT_BRANCH` / `PROD_BRANCH` | per repo *(declare each in the real profile)* |
| `WORKTREE_ROOT` | per repo, declared **once** and matched exactly by the sweep — e.g. `../core-api-wt/<id>` |
| `BUILD_CMD` · `TEST_CMD` · `FULL_SUITE_CMD` · `LINT_CMD` · `SMOKE_CMD` | per repo *(declare the concrete commands in the real profile; each child PBI verifies to its own repo's bar)* |
| `LANG_CONVENTIONS` · `INVARIANTS` · `DECISION_RECORD` · `DOCS_SINKS` | per repo / project *(declare in the real profile)* |
| `DESIGN_ARTIFACTS` | version-controlled storyboards: real page context, affected views/states, responsiveness, accessibility |
| `KNOWN_ENV_LIMITS` · `DEP_AUDIT_CMD` · `MIGRATIONS_DIR` (+ down/up/dump) · `DOCS_ROOT` · `DOC_DOMAINS` · `TOKEN_TOOL` | *(declare or leave unbound)* |

## Variation points

- **Spine:** full 9-stage, explicit tail —
  `intake → design → queued → build → verify → audit → ready → shipping → complete`. Gates:
  `queued` and `ready` (Feature-level, human-only).
- **VP1 tracker:** exactly as the binding. Routing tags in use: `repo:core-api`, `repo:webapp`,
  `repo:utility`.
- **VP2 topology:** multi-repo Feature/child per the binding § 1.
- **VP3 modules:**
  - **Design UX track: bound**, per child. Intake classifies frontend impact none/minor/material;
    material changes require the storyboards above and a human `design-approve`/`design-reject`,
    recorded as child design evidence. Design approval does not replace the `queued` gate.
  - **PSI lane: on.** `reported → triage → diagnosed → decided → pending-fix → resolved`. PSI
    worker is read-only (severity, repro, affected surfaces, evidence, root-cause hypothesis; no
    code, no branches). At `decided` a human chooses no-change / documentation / duplicate /
    needs-code; needs-code creates or attaches to a Feature. Priority 1 does **not** bypass
    automation.
- **VP4 dispatcher:** interactive GitHub Copilot session; scheduler mechanism TBD. No dispatcher
  singleton — overlapping sessions deconflict via per-item work locks and idempotent tracker
  writes; machine-local Git/PR maintenance is serialized by a workspace lock file (30-min stale
  reap). Reaps stale work locks (2h heuristic, verify-before-write), coordinates Features, fans
  out repository workers, performs a PSI pass, posts a digest. Worker isolation: per-repo
  issue-scoped worktrees at the declared patterns.
- **VP5 quality bars:** per repo (`core-api`, `webapp`, `utility`); each child verifies to its own
  repo's full-verification + smoke bar.
- **Deterministic core:** `sdlc.ps1` — owns tag mutation, CAS claim/release, gate checks, the
  needs-human ritual, and the status-block rewrite. Operational reference: `SDLC-PIPELINE.md`;
  worker contract: local `README.md` + `dispatch.md` transcribed from this repo.

## Known deviations from spec

- Child PBIs can report `result:build-blocked` but never own `sdlc:needs-human` — human
  attention is consolidated at the Feature (spec allows either; declared for clarity).
- Two-hour stale-lock heuristic retained despite reap-and-retry churn; workers must leave a
  clear outcome note (in the stage Task) before releasing locks (lesson of an opaque production
  block: lock state without a reason is undebuggable).
- Claim comments and the `sdlc:wip-<run-id>` tag family are retired: the ownership record is
  the stage-Task claim line (rev-CAS), and `sdlc:wip` is a pure visibility tag removed on
  release. The spec's `gh-issue` binding keeps label + comment; declared for clarity.
- `Custom.SdlcStage` / `Custom.SdlcLock` fields retired — the stage marker is a Feature tag and
  the lock substrate is the stage Task, so no process-template admin is required.
- PBI Done fires on PR raised, not PR merged — merge, verify, and audit are Feature-level
  concerns; a Done PBI is never reopened, and post-Done rework is a new estimated PBI.
- Description status block is a declared **cache** outside the evidence contract — it carries
  no authority and may be regenerated at any time.
