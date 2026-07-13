# SDLC conformance profile: work (Azure DevOps) — worked example

A reference transcription of a read-only-consumer fork: the framework repo cannot be installed
from; agents read this spec and this profile and maintain the local implementation by hand.
Bindings per [Composability.md](../Composability.md).

- **Spine:** full 9-stage, explicit tail —
  `intake → [design] → queued → build → verify → audit → ready → shipping → complete`.
  Gates: `delivery:queued` and `delivery:ready` (Feature-level, human-only).

- **VP1 tracker: Azure DevOps work items.**
  | Abstract operation | Binding |
  |---|---|
  | stage marker | Feature phase (state machine above); child phase evidence `result:build-passed` / `result:build-blocked` |
  | routing marker | exactly one `repo:*` tag per child PBI (`repo:core-api`, `repo:webapp`, `repo:utility`) |
  | claim lock | `sdlc:wip` and `sdlc:wip-<run-id>` tags on the child |
  | park to human | `sdlc:needs-human` **on the parent Feature only** + `HUMAN ACTION REQUIRED` discussion comment naming the blocked child; blocks all automated work on that Feature |
  | human keep-off | `sdlc:hold` |
  | evidence record | child PBI: technical plan, branch/PR, phase evidence; Feature: gates + ACs |
  | hierarchy & ordering | ADO Parent link = membership; predecessor/successor links = provider→consumer order (dependency graph lives in ADO relationships, not tags) |
  | tag mutation safety | **all tag ops via `sdlc.ps1`** — raw ADO CLI tag updates append rather than safely replace the tag string |

- **VP2 topology: multi-repo Feature/child.** Feature is the authoritative record (lifecycle,
  requirements, ACs, dependencies, gates, integration, release); child PBIs are subordinate
  execution records with no lifecycle of their own. Providers and independent children build
  first; a consumer starts once its predecessor publishes `contract-ready` evidence (need not wait
  for the backend PR to ship). Verify = per-child full verification; audit reconciles security,
  invariants, contracts, PRs, docs, and merge order across children; `complete` only when all
  children shipped and Feature ACs pass.

- **VP3 modules:**
  - **Design lane: on**, per child. Intake classifies frontend impact none/minor/material;
    material changes require version-controlled storyboards (real page context, affected
    views/states, responsiveness, accessibility) and a human `design-approve`/`design-reject`,
    recorded as child design evidence. Design approval does not replace the `queued` gate.
  - **PSI lane: on.** `reported → triage → diagnosed → decided → pending-fix → resolved`. PSI
    worker is read-only (severity, repro, affected surfaces, evidence, root-cause hypothesis; no
    code, no branches). At `decided` a human chooses no-change / documentation / duplicate /
    needs-code; needs-code creates or attaches to a Feature. Priority 1 does **not** bypass
    automation.

- **VP4 dispatcher:** interactive GitHub Copilot session; scheduler mechanism TBD. Singleton lock
  acquired per session; reaps stale work locks (2h heuristic), does Git/PR maintenance,
  coordinates Features, fans out repository workers, performs a PSI pass, posts a digest,
  releases the lock. Workers follow the narrow CLAIM → WORK → EMIT → STOP contract; `continue`
  releases the lock without a terminal phase result.

- **VP5 quality bars:** per repo (`core-api`, `webapp`, `utility`); each child verifies to its own
  repo's full-verification + smoke bar. *(Declare the concrete commands in the real profile.)*

- **Deterministic core:** `sdlc.ps1` (local implementation; owns tag mutation, gate checks,
  needs-human ritual). Operational reference: `SDLC-PIPELINE.md`; worker contract: `README.md` +
  `dispatch.md` (local).

- **Known deviations from spec:**
  - Child PBIs can report `result:build-blocked` but never own `sdlc:needs-human` — human
    attention is consolidated at the Feature (spec allows either; declared for clarity).
  - Two-hour stale-lock heuristic retained despite reap-and-retry churn; workers must write a
    clear outcome comment before releasing locks (lesson of the opaque #270410 block: tag state
    without a reason is undebuggable).
