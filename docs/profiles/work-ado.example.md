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
  | claim lock | `Custom.SdlcLock` field via rev-CAS (preferred) — fallback: single `sdlc:wip` tag + claim comment; see **Locking + status model** below |
  | park to human | `sdlc:needs-human` **on the parent Feature only** + `HUMAN ACTION REQUIRED` discussion comment naming the blocked child; blocks all automated work on that Feature |
  | human keep-off | `sdlc:hold` |
  | evidence record | child PBI: technical plan, branch/PR, phase evidence; Feature: gates + ACs |
  | hierarchy & ordering | ADO Parent link = membership; predecessor/successor links = provider→consumer order (dependency graph lives in ADO relationships, not tags) |
  | tag mutation safety | **all tag ops via `sdlc.ps1`** — raw ADO CLI tag updates append rather than safely replace the tag string |
  | status dashboard | description status block on Feature + children — a dispatcher-rewritten **cache**, never a record (format below) |

- **Locking + status model (VP1 detail).** Two sanctioned lock variants — the live profile
  declares exactly one, chosen by whether inherited-process (process-template) admin is
  available. Both satisfy the spec's lock-substrate contract
  ([Composability.md](../Composability.md#vp1--tracker-backend)).

  - **Preferred — custom fields** *(needs process-template admin)*: two fields on Feature and
    child PBI types — `Custom.SdlcStage` (the stage marker) and `Custom.SdlcLock`, holding
    `<run-id> <ISO 8601 claim time>`; empty = unlocked. Every write goes through `sdlc.ps1` as a
    REST `PATCH` whose JSON Patch body **leads with** `{"op":"test","path":"/rev","value":<rev
    read with the item>}` — a true compare-and-swap: a concurrent writer bumps the rev, the test
    fails, the loser re-reads and either backs off (lock now held) or retries (unrelated edit).
    CAS serializes claims at the tracker, so the template's claim-verify ritual and
    timeline-boundary computation are **unnecessary in this variant** — claim = one conditional
    write. WIQL on the field gives the dispatcher's Step 0 "all locked items" snapshot; the
    field's revision history is the server-side proof of age and owner.
  - **Fallback — slim tag + comment** *(no process admin)*: a **single** `sdlc:wip` tag, carrying
    no information beyond visibility (board filters, WIQL snapshot), plus a claim discussion
    comment `sdlc:claim <run-id> <lane>` as the ownership record and race tiebreaker. The run-id
    lives **only** in the comment — the `sdlc:wip-<run-id>` tag family is retired (it duplicated
    the comment and generated tag churn ADO's string-typed tag field handles badly). Claim-verify
    runs per the template `prompts/sdlc/README.md`; the boundary analog for "newest `sdlc:wip`
    *unlabeled* timeline event" is **the work-item revision in which the tag disappeared** —
    ADO revision history is the timeline-event equivalent, queried via
    `workItems/<id>/revisions` and diffed on `System.Tags`.
  - **Reaping (either variant):** age and owner must be **server-side provable** — the claim
    comment's own timestamp, or the lock field's revision — never a worker-authored string alone,
    and never `System.ChangedDate` (any later edit refreshes it, exactly like GitHub
    `updatedAt`). Unprovable age → leave the lock and record it; never reap. The 2h threshold and
    verify-before-write re-check carry over from the template unchanged.
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
    (comments, fields, PR state), splicing only between the sentinels and never regenerating the
    rest of the description. Because it is derived state, mangling is costless: a parse failure
    or human edit inside the block → regenerate next cycle from evidence; never trust it, never
    repair it by guesswork, and **never let a lock live only here**. Its job is to spare humans
    and agents from trawling the comment stream for current state — it removes read load, not
    the record.

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
  singleton — overlapping sessions deconflict via per-issue work locks and idempotent tracker
  writes; machine-local Git/PR maintenance is serialized by a workspace lock file (30-min stale
  reap). Reaps stale work locks (2h heuristic, verify-before-write), coordinates Features, fans
  out repository workers, performs a PSI pass, posts a digest. Workers follow the narrow CLAIM →
  WORK → EMIT → STOP contract; `continue` releases the work lock without a terminal phase result.
  Worker isolation: per-repo issue-scoped worktrees; each repo declares its worktree path pattern
  **once**, and the maintenance sweep matches exactly that pattern — the creation pattern and the
  sweep pattern are a declared pair, never allowed to drift independently. *(Declare the concrete
  patterns per repo in the real profile, e.g. `../core-api-wt/<id>`.)*

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
  - `sdlc:wip-<run-id>` tags retired: the run-id is carried only in the claim comment (fallback
    variant) or the lock field (preferred variant). The spec's GitHub binding keeps label +
    comment; declared for clarity.
  - Description status block is a declared **cache** outside the evidence contract — it carries
    no authority and may be regenerated at any time.
