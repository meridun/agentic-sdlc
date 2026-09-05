# Binding: `ado-feature` — Azure DevOps, Feature / PBI / Task, multi-repo

The multi-repo shape: a **Feature** is the deployable unit and authoritative record; child
**PBIs** are per-repo execution records; **stage Tasks** carry the agent's locks and working
memory. Distilled from a production read-only-consumer fork (the framework repo could not be
installed from; agents transcribed this contract into local Copilot agent files and an
`sdlc.ps1`). Fills the contract in [`../README.md`](../README.md). The single-repo degenerate form
is [`../ado-pbi/BINDING.md`](../ado-pbi/BINDING.md).

## 1. Unit of work

The **Feature** (work item id `#<n>`). Three levels, three jobs — agents respect all three:

| Level | Job | Description holds | Native states |
|---|---|---|---|
| **Feature** | the deployable unit and authoritative record | business requirements + acceptance criteria (+ status-block cache) | New, In Progress, On Hold, Done, Removed |
| **PBI** | repo-specific block of work, effort-estimated | technical specification + implementation plan | New, Approved, Committed, On Hold, Done, Removed |
| **Task** | actual work performed, per SDLC stage | claim record, stage working context, notes; Estimated/Remaining/Completed hours | New, In Progress, On Hold, Done, Removed |

Topology (`docs/Composability.md` VP2): providers and independent children build first; a consumer
may start once its predecessor publishes `contract-ready` evidence (it need not wait for the
provider's PR to ship). At `verify` every child runs its own repo's full verification; `audit`
reconciles across children; `complete` requires all children shipped **and** Feature-level ACs
pass. A child PBI never owns human-attention state.

## 2. Spine form

**Explicit tail** — `intake → design → queued → build → verify → audit → ready → shipping →
complete`, gates at `queued` and `ready` (Feature-level, human-only).

## 3. Marker substrate — tags on the Feature

| Marker | Binding |
|---|---|
| `stage:<name>` | tag on the **Feature**, exactly one, replaced on advance. Feature native State is **derived**: In Progress ⇔ stage ≥ build; Done ⇔ complete |
| routing | exactly one `repo:*` tag per child PBI (`repo:core-api`, …) |
| `sdlc:wip` | transient tag on the claimed Feature/PBI — **visibility only** (board filters, the WIQL snapshot); removed on lock release |
| `sdlc:needs-human` | on the **parent Feature only** + a `HUMAN ACTION REQUIRED` discussion comment naming the blocked child; blocks all automation on that Feature |
| `sdlc:hold` | tag on the Feature |
| priority | ADO `Priority` field (1 › 2 › 3 › 4), then `CreatedDate` FIFO |

**Tag budget:** titles carry bracketed reporting keywords, so the only sanctioned tag families are
`stage:*`, `repo:*`, transient `sdlc:wip`, `sdlc:hold`, `sdlc:needs-human`. No `sdlc:wip-<run-id>`
family, no custom `Custom.SdlcStage` / `Custom.SdlcLock` fields — nothing needs process-template
admin. **All tag mutation goes through the deterministic core** — raw ADO CLI tag updates
append/replace the whole tag string unsafely.

## 4. Lock substrate — compare-and-swap on the stage Task

- **Claim.** The worker lazily creates (or finds) the **stage Task** for the stage it is about to
  work — parented to the PBI for repo work, to the Feature for feature-level stages — and writes
  the claim as the **first line of the Task description**: `sdlc:claim <run-id> <ISO 8601>`,
  setting the Task State to In Progress in the same PATCH. The PATCH **leads with**
  `{"op":"test","path":"/rev","value":<rev read with the item>}` — a true CAS: a concurrent
  writer bumps the rev, the test fails, the loser re-reads and backs off (lock now held) or
  retries (unrelated edit). CAS serializes claims at the tracker, so the core's claim-verify
  ritual and settled/live boundary are **unnecessary** — claim = one conditional write. Then add
  `sdlc:wip` to the claimed item for visibility.
- **Release.** Clear the claim line, set Remaining/Completed hours, remove `sdlc:wip`. On stage
  advance the Task goes Done; on CONTINUE it stays open and its description carries the handoff
  context for the next session.
- **Age and owner for the reaper** come from the Task **revision** in which the claim line was
  written (`workItems/<id>/revisions`) — that proves both when and by which identity. Never trust
  the worker-authored timestamp string alone, never `System.ChangedDate` (any later edit refreshes
  it, exactly like GitHub `updatedAt`). Unprovable → leave it, record it. The 2 h threshold and
  verify-before-write carry over unchanged. A reap clears the claim line and removes `sdlc:wip`;
  the note goes in the Task, not a Feature comment.

**Comment budget.** Comments communicate across sessions and to the team — phase results, gate
requests, blocks, human discussion — posted on the **Feature** even when the work happened on a
PBI or Task. Lock traffic (claims, releases, healthy reaps) produces **no comments**.

## 5. Evidence locations

| Canonical section | Lives in |
|---|---|
| *(original author text)*, `## Requirements`, `## Acceptance criteria` | **Feature** description |
| `## Design`, `## Implementation plan` | **child PBI** description (one per repo; the Feature's plan is the ordered set of children) |
| stage working memory, claim record, hours | the **stage Task** (created lazily when the stage begins, Done when it advances; skipped stages get none, so the Task trail doubles as the stage history) |
| comment channel | **Feature** discussion |

**Status block — a cache, never a record.** A fixed-format block at the top of the Feature and
child descriptions, delimited by plain-text sentinels (the rich-text editor strips HTML comments):

```text
[SDLC-STATUS]
stage: build · lock: none · updated: 2026-07-15T18:40Z
branch: feat/1234-export · pr: !5678 (open)
last: BUILD ADVANCE(verify) — targeted tests green
[/SDLC-STATUS]
```

The dispatcher rewrites it idempotently each cycle **from the evidence record**, splicing only
between the sentinels. A parse failure or human edit → regenerate; never trust it, never repair it
by guesswork, and **never let a lock live only here**.

## 6. Dependency model — ADO links

Parent link = membership. **Predecessor/successor** links = provider → consumer order; the
dispatcher's gate reads them (`dep-read` = WIQL/REST relations of type
`System.LinkTypes.Dependency-Reverse`). A blocker outside the project or organization can't be a
link → `sdlc:hold` + prose. Within a Feature, ordered children are linked the same way; a child is
also unblocked by its predecessor's `contract-ready` evidence comment, not only by its closure.

## 7. Code host — ADO PRs

Per child PBI, one branch and one PR in that PBI's repo (`!<id>`), linked via the PR's work-item
association; `pr-list` / `pr-state` = `az repos pr list --status active` / `az repos pr show`.
**PBI state machine** (agents never skip a human-owned transition):

| Transition | Owner | Meaning |
|---|---|---|
| New → Approved | human | requirements + ACs inherited from the Feature are set — the PBI-level reflection of the Feature's `queued` gate |
| Approved → Committed | human | sprint planning: which PBIs commit now |
| Committed → Done | agent | **the PR is raised** — code complete incl. unit tests; does not wait for merge |
| any → On Hold / Removed | human | keep-off / cancellation |

Agents treat Approved and Committed as workable (Committed first). **Done PBIs are never
reopened**: post-Done findings (audit failures, review rework, scope changes) become a **new PBI**
under the same Feature, re-estimated. Feature-level verify/audit/ship continue independently of
child PBI Done. Iteration/sprint assignment is meaningful only on PBIs.

## 8. Deterministic core — `sdlc.ps1` (local)

The production fork carries its own `sdlc.ps1` implementing the operations below (tag mutation,
CAS claim/release, gate checks, needs-human ritual, status-block rewrite). It is **not shipped
here** (read-only consumer); a fork that can install writes one against this table, keeping the
same operation names. `<SDLC_CLI>` = `pwsh ./sdlc.ps1` in that fork.

## 9. Operation table

| Op | Binding |
|---|---|
| `snapshot` | one WIQL query: open Features with tags, Priority, CreatedDate, Parent/Predecessor relations; children with `repo:*` tags |
| `read <issue>` | Feature description + discussion; children's descriptions; the current stage Task |
| `history <issue>` | Feature discussion comments (phase results) + stage Task revisions (claims) |
| `dep-read` | predecessor/successor relations (§6) + `contract-ready` evidence comments |
| `dup-search` | WIQL `[System.Title] CONTAINS` over open Features/PBIs; judgment stays with the caller |
| `in-flight <stage>` | WIQL `Tags CONTAINS 'stage:<stage>'` |
| `closed-since` | WIQL on `ClosedDate` in window → successors of each |
| `pr-list` / `pr-state` | §7 |
| `lock-age` | stage Task's claim-line revision (§4) |
| `claim` / `next` | §4 CAS PATCH via the core; `next` orders by Priority then CreatedDate |
| `emit` | Feature comment with `sdlc:emit <run-id> <OUTCOME>` first line; stage tag swap (graph-validated by the core); Task release; `sdlc:needs-human` on PARK; native State derivation |
| `comment` | Feature discussion |
| `write-section` | PATCH the owning description (Feature or PBI), splicing only the section |
| `dep-edge` | add a predecessor/successor link |
| `file` | create a Feature (stage:intake tag) — or a PBI under it |
| `close` | Feature State → Done (only at `complete`) or Removed (intake CLOSE) |
| `pr-open` | `az repos pr create` in the child's repo, linked to the PBI |
| `reap` | §4 — Task claim line cleared + `sdlc:wip` removed, note in the Task |
| `advance` | stage tag swap via the core |
| `stage-repair` | add `stage:intake` tag via the core |
| `park` | `sdlc:needs-human` tag on the Feature + `HUMAN ACTION REQUIRED` comment |
| `readiness-derive` | the status-block rewrite (cache) — no derived tags |
| `sweep-ack` | none — window-bounded |

## 10. Gotchas

- **Raw tag writes clobber.** Only the core mutates tags.
- **`ChangedDate` is not lock age.** Use the claim-line revision.
- **Rich-text descriptions strip HTML comments** — status-block sentinels must be visible text.
- **Child PBIs never carry `sdlc:needs-human`**; human attention consolidates on the Feature
  (they may report `result:*` evidence).
- **Lock state without a reason is undebuggable** (lesson of an opaque production block): a
  worker leaves an outcome note in the stage Task before releasing.
- Each repo declares its worktree path pattern **once**; the maintenance sweep matches exactly
  that pattern (creation pattern and sweep pattern are a declared pair).
