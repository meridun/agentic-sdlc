# SDLC profile: monotool — worked example (`ado-pbi`)

> **Proposed.** The `ado-pbi` binding has no production lineage yet; this example shows what a
> first adoption's profile would declare. Validate against a real project and port what you learn
> into [`ado-pbi/BINDING.md`](../../sdlc/bindings/ado-pbi/BINDING.md).

A single ADO repo (a mono-repo or a small tool) where **one PBI is a shippable unit**: no
Features in the pipeline's model, stage Tasks for locks and working memory, one ADO PR per PBI.
Bindings per [Composability.md](../Composability.md).

Spec version: `v2.0.0`

## Keys

| Key | Value |
|---|---|
| `BINDING` | `ado-pbi` |
| `SDLC_CLI` | `pwsh ./sdlc.ps1` (the `ado-feature` core with the Feature level dropped) |
| `SPEC_VERSION` | `v2.0.0` |
| `PROJECT` | `monotool` |
| `REPO_PATH` | `C:\work\monotool` |
| `WORKER_AGENT` | `monotool-sdlc-worker` |
| `DEFAULT_BRANCH` | `develop` |
| `PROD_BRANCH` | `main` |
| `WORKTREE_ROOT` | `../monotool-wt` |
| `BUILD_CMD` | `dotnet build` |
| `TEST_CMD` | `dotnet test --filter <name>` |
| `FULL_SUITE_CMD` | `dotnet test` |
| `LINT_CMD` | `dotnet format --verify-no-changes` |
| `SMOKE_CMD` | `pwsh ./scripts/smoke.ps1` |
| `LANG_CONVENTIONS` | `dotnet format` clean, analyzers at warning-as-error, xUnit green |
| `INVARIANTS` | *(declare)* |
| `DECISION_RECORD` | `docs/Decisions.md` |
| `DOCS_SINKS` | `README.md`, `docs/` |
| optional keys | *(unbound unless declared)* |

## Variation points

- **Spine:** collapsed tail — ship opens the ADO PR, the human merge is the `ready` gate; PBI
  native State → Done at merge (via intake's close sweep).
- **VP1 tracker:** exactly as the binding — `stage:*` and the control markers are tags on the
  PBI; rev-CAS lock on the stage Task; every canonical section in the PBI description;
  predecessor/successor links between PBIs.
- **VP2 topology:** single repo — the PBI is the Feature and its only child.
- **VP3 modules:** design UX track *(off unless the tool has UI)*; PSI lane off.
- **VP4 dispatcher:** *(declare: scheduled Copilot / Claude Code task; workspace lock file;
  worktrees at `../monotool-wt/<id>`)*.
- **VP5 quality bars:** the commands above.
- **Deterministic core:** `sdlc.ps1`.

## Known deviations from spec

- PBI native **Done fires on merge**, not on PR raised (differs from `ado-feature`, where the
  Feature owns the lifecycle) — stated in the binding; repeated here because a drift audit
  against `ado-feature` would otherwise flag it.
