# SDLC conformance profile: work (Azure DevOps) — worked example

A reference transcription of a read-only-consumer fork: the framework repo cannot be installed
from; agents read this spec and this profile and maintain the local implementation by hand.
Bindings per [Composability.md](../Composability.md).

- **Spine:** full 9-stage, explicit tail —
  `intake → [design] → queued → build → verify → audit → ready → shipping → complete`.
  Gates: `delivery:queued` and `delivery:ready` (Feature-level, human-only).

- **VP1 tracker: Azure DevOps work items — Feature / PBI / Task hierarchy.**

  The three work-item levels have distinct jobs; agents respect all three:

  | Level | Job | Description holds | States (native ADO) |
  |---|---|---|---|
  | **Feature** | the deployable unit and authoritative record | business requirements + acceptance criteria (+ status-block cache) | New, In Progress, On Hold, Done, Removed |
  | **PBI** | repo-specific block of work, effort-estimated | technical specification + implementation plan | New, Approved, Committed, On Hold, Done, Removed |
  | **Task** | actual work performed, per SDLC stage | claim record, stage working context, implementation notes; Estimated/Remaining/Completed hours | New, In Progress, On Hold, Done, Removed |

  | Abstract operation | Binding |
  |---|---|
  | stage marker | `stage:<stage>` tag on the **Feature** — exactly one, replaced on advance. ADO Feature State is **derived**, never authoritative: In Progress ⇔ development has actively begun (stage ≥ build); Done ⇔ complete |
  | routing marker | exactly one `repo:*` tag per child PBI (`repo:core-api`, `repo:webapp`, `repo:utility`) |
  | claim lock | transient `sdlc:wip` tag (visibility only) + **stage-Task claim record** written by rev-CAS; see **Locking + status model** below |
  | park to human | `sdlc:needs-human` **on the parent Feature only** + `HUMAN ACTION REQUIRED` discussion comment naming the blocked child; blocks all automated work on that Feature |
  | human keep-off | `sdlc:hold` |
  | evidence record | Feature discussion comments (phase results, gates, blocks — all team-facing traffic rolls up here, even when the work happened on a child); PBI description (tech spec / plan); stage Tasks (working context, notes, hours) |
  | hierarchy & ordering | ADO Parent link = membership; predecessor/successor links = provider→consumer order (dependency graph lives in ADO relationships, not tags) |
  | tag mutation safety | **all tag ops via `sdlc.ps1`** — raw ADO CLI tag updates append rather than safely replace the tag string |
  | status dashboard | description status block on Feature + children — a dispatcher-rewritten **cache**, never a record (format below) |

  **Tag budget.** Work-item titles carry bracketed reporting keywords, so tags stay minimal:
  the only sanctioned families are `stage:*` (one, Feature only), `repo:*` (one per PBI),
  transient `sdlc:wip` (removed on lock release), and the human controls `sdlc:hold` /
  `sdlc:needs-human`. No other tags; the `sdlc:wip-<run-id>` family and the `Custom.SdlcStage` /
  `Custom.SdlcLock` fields are retired.

  **Comment budget.** Comments are for communicating across sessions and to the team: phase
  results, gate requests, blocks, and human discussion — posted on the **Feature** even when the
  underlying work happened on a PBI or Task. Lock traffic (claims, releases, reap notes on
  healthy reaps) produces **no comments**; it lives in the stage Task.

- **Locking + status model (VP1 detail).** One lock design, no process-template admin required.
  Satisfies the spec's lock-substrate contract
  ([Composability.md](../Composability.md#vp1--tracker-backend)).

  - **Claim.** The worker lazily creates (or finds) the **stage Task** for the stage it is about
    to work — parented to the PBI for repo work, to the Feature for feature-level stages — and
    writes the claim as the **first line of the Task description**: `sdlc:claim <run-id>
    <ISO 8601 claim time>`, setting the Task State to In Progress in the same PATCH. The PATCH
    **leads with** `{"op":"test","path":"/rev","value":<rev read with the item>}` — a true
    compare-and-swap: a concurrent writer bumps the rev, the test fails, the loser re-reads and
    backs off (lock now held) or retries (unrelated edit). CAS serializes claims at the tracker,
    so the template's claim-verify ritual and timeline-boundary computation are **unnecessary** —
    claim = one conditional write. The worker then adds the `sdlc:wip` tag to the claimed
    Feature/PBI via `sdlc.ps1`; the tag carries no information beyond visibility (board filters,
    the dispatcher's Step 0 WIQL snapshot).
  - **Release.** Clear the claim line, set Remaining/Completed hours, remove `sdlc:wip`. On stage
    advance the Task also goes to Done; on `continue` (no terminal phase result) the Task stays
    open and its description carries the handoff context for the next session.
  - **Reaping.** Age and owner must be **server-side provable**: the Task revision in which the
    claim line was written (queried via `workItems/<id>/revisions`) proves both when and by which
    identity — never trust the worker-authored timestamp string alone, and never
    `System.ChangedDate` (any later edit refreshes it, exactly like GitHub `updatedAt`).
    Unprovable age → leave the lock and record it; never reap. The 2h threshold and
    verify-before-write re-check carry over from the template unchanged. A reap clears the claim
    line and removes `sdlc:wip`; the outcome note goes in the Task, not a Feature comment.
  - **Status block — a cache, never a record.** A fixed-format block at the **top** of the
    description (Feature and child), delimited by plain-text sentinels — the ADO rich-text editor
    strips HTML comments on save, so the markers must be visible text:

    ```text
    [SDLC-STATUS]
    stage: build · lock: none · updated: 2026-07-15T18:40Z
    branch: feat/1234-export · pr: !5678 (open)
    last: BUILD ADVANCE(verify) — targeted tests green
    [/SDLC-STATUS]
    ```

    The dispatcher rewrites it idempotently each cycle, **derived from the evidence record**
    (comments, Tasks, PR state), splicing only between the sentinels and never regenerating the
    rest of the description. Because it is derived state, mangling is costless: a parse failure
    or human edit inside the block → regenerate next cycle from evidence; never trust it, never
    repair it by guesswork, and **never let a lock live only here**. Its job is to spare humans
    and agents from trawling the comment stream for current state — it removes read load, not
    the record.

- **Stage Tasks (VP1 detail).** Created **lazily** when a stage begins, marked Done when the
  stage advances; skipped stages get no Task, so the Task trail doubles as the stage history.
  Humans rarely read them — they are the agent's durable working memory: claim record (above),
  stage context and implementation notes that must survive across sessions, and hours. Workers
  set Estimated Hours on creation, keep Remaining/Completed current on release. Anything a
  teammate needs to know graduates from the Task to a Feature comment; anything the next session
  needs stays in the Task.

- **PBI state machine (VP1 detail).** Native ADO PBI states carry the workflow; agents never
  skip a human-owned transition:

  | Transition | Owner | Meaning |
  |---|---|---|
  | New → Approved | human | requirements + ACs inherited from the Feature are set; development may begin — the PBI-level reflection of the Feature's `queued` gate |
  | Approved → Committed | human | at sprint planning, when the Feature hits `stage:queued`, the human decides which PBIs commit to the current sprint |
  | Committed → Done | agent | **the PR is raised** — code complete including unit tests, tests run outside local; does not wait for merge |
  | any → On Hold / Removed | human | keep-off / cancellation |

  Agents treat Approved and Committed as workable (Committed first). **Done PBIs are never
  reopened**: post-Done findings — audit failures, PR-review rework, scope changes — become a
  **new PBI** under the same Feature, re-estimated. Feature-level verify/audit/ship continue
  independently of child PBI Done. Iteration/sprint assignment is meaningful only on PBIs
  (via Committed); Features ignore it.

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

- **VP4 dispatcher:** interactive GitHub Copilot session; scheduler mechanism TBD. No dispatcher
  singleton — overlapping sessions deconflict via per-item work locks and idempotent tracker
  writes; machine-local Git/PR maintenance is serialized by a workspace lock file (30-min stale
  reap). Reaps stale work locks (2h heuristic, verify-before-write), coordinates Features, fans
  out repository workers, performs a PSI pass, posts a digest. Workers follow the narrow CLAIM →
  WORK → EMIT → STOP contract; `continue` releases the work lock without a terminal phase result,
  leaving handoff context in the open stage Task. Worker isolation: per-repo issue-scoped
  worktrees; each repo declares its worktree path pattern **once**, and the maintenance sweep
  matches exactly that pattern — the creation pattern and the sweep pattern are a declared pair,
  never allowed to drift independently. *(Declare the concrete patterns per repo in the real
  profile, e.g. `../core-api-wt/<id>`.)*

- **VP5 quality bars:** per repo (`core-api`, `webapp`, `utility`); each child verifies to its own
  repo's full-verification + smoke bar. *(Declare the concrete commands in the real profile.)*

- **Deterministic core:** `sdlc.ps1` (local implementation; owns tag mutation, gate checks,
  needs-human ritual). Operational reference: `SDLC-PIPELINE.md`; worker contract: `README.md` +
  `dispatch.md` (local).

- **Known deviations from spec:**
  - Child PBIs can report `result:build-blocked` but never own `sdlc:needs-human` — human
    attention is consolidated at the Feature (spec allows either; declared for clarity).
  - Two-hour stale-lock heuristic retained despite reap-and-retry churn; workers must leave a
    clear outcome note (in the stage Task) before releasing locks (lesson of the opaque #270410
    block: lock state without a reason is undebuggable).
  - Claim comments and the `sdlc:wip-<run-id>` tag family are retired: the ownership record is
    the stage-Task claim line (rev-CAS), and `sdlc:wip` is a pure visibility tag removed on
    release. The spec's GitHub binding keeps label + comment; declared for clarity.
  - `Custom.SdlcStage` / `Custom.SdlcLock` fields retired — the stage marker is a Feature tag and
    the lock substrate is the stage Task, so no process-template admin is required.
  - PBI Done fires on PR raised, not PR merged — merge, verify, and audit are Feature-level
    concerns; a Done PBI is never reopened, and post-Done rework is a new estimated PBI.
  - Description status block is a declared **cache** outside the evidence contract — it carries
    no authority and may be regenerated at any time.
