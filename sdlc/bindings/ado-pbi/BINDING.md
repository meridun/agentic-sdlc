# Binding: `ado-pbi` — Azure DevOps, PBI / Task, single repo

> **Status: proposed.** This is the single-repo degenerate form of
> [`ado-feature`](../ado-feature/BINDING.md) (`docs/Composability.md` VP2: "the issue is
> simultaneously the Feature and its only child") for mono-repos and smaller projects where **a
> single PBI is shippable**. It has no production lineage yet. Everything not stated here is
> inherited from `ado-feature` verbatim; the first adoption should validate the deltas below and
> port what it learns back to this file.

## 1. Unit of work

The **PBI** (work item id `#<n>`). Two levels:

| Level | Job | Description holds | Native states |
|---|---|---|---|
| **PBI** | the shippable unit and authoritative record | original text + every canonical artifact section (+ status-block cache) | New, Approved, Committed, Done, Removed |
| **Task** | actual work performed, per SDLC stage | claim record, stage working context, notes, hours | New, In Progress, Done, Removed |

No Features in the pipeline's model (a Feature may exist above as a human grouping; the pipeline
neither reads nor writes it). No `repo:*` routing tags, no `contract-ready` handshake.

## 2. Spine form

**Collapsed tail** by default — ship opens the ADO PR, the human merge is the `ready` gate,
`shipping → complete` fold into merge-and-Done. A fork with a real release step declares the
explicit tail in its profile; the lane prompts are unchanged either way.

## 3. Marker substrate — tags on the PBI

| Marker | Binding |
|---|---|
| `stage:<name>` | tag on the **PBI**, exactly one, replaced on advance |
| `sdlc:wip` / `sdlc:hold` / `sdlc:needs-human` | tags on the **PBI** (there is no parent to consolidate onto) |
| priority | ADO `Priority` field, then `CreatedDate` FIFO |

**Native PBI State is derived, never authoritative:** New while stage ≤ design; **Approved** is
the human's `queued`-gate admission (the human sets it when admitting `stage:queued →
stage:build`; automation treats Approved/Committed as the same admission signal); Done ⇔ the PR
merged (the terminal event). This differs from `ado-feature`, where PBI Done fires on *PR raised*
— there the Feature owns the lifecycle; here the PBI does, so Done must mean shipped. Post-Done
rework is still a **new PBI**, never a reopen.

## 4. Lock substrate

Inherited: **compare-and-swap on the stage Task**, parented to the PBI, claim line first in the
Task description, `/rev` test leading the PATCH, age/owner from the claim-line revision.

## 5. Evidence locations

| Canonical section | Lives in |
|---|---|
| *(original author text)*, `## Requirements`, `## Acceptance criteria`, `## Design`, `## Implementation plan` | the **PBI** description — same layout as a GitHub issue body, one owner per section |
| claim record, stage working memory, hours | the **stage Task** |
| comment channel | **PBI** discussion |

Status block: inherited (cache on the PBI description).

## 6. Dependency model

Inherited: predecessor/successor links **between PBIs**. A Feature-level parent link is ignored
by the gate.

## 7. Code host

One branch, one ADO PR (`!<id>`) per PBI in the single repo, linked to the PBI; merge → PBI Done.
The human-owned transitions are New → Approved (queued gate) and any → Removed; the agent owns
nothing on the native State except deriving Done at merge (via the intake close sweep, since the
merge fires no worker).

## 8. Deterministic core

Inherited: a local `sdlc.ps1` written against the operation table, or none. The `ado-feature`
core adapts by dropping the Feature level (stage tag and human markers move to the PBI; `snapshot`
queries PBIs).

## 9. Operation table — deltas from `ado-feature`

| Op | Delta |
|---|---|
| `snapshot` | WIQL over open **PBIs** with tags, Priority, CreatedDate, predecessor relations |
| `read` / `write-section` | the PBI description holds every section |
| `history` | PBI discussion + stage Task revisions |
| `emit` | comment on the PBI; stage tag swap on the PBI; `sdlc:needs-human` on the PBI |
| `park` | on the PBI |
| `close` | PBI State → Removed (intake CLOSE); Done only at merge |
| `pr-open` | in the single repo, linked to the PBI |
| everything else | as `ado-feature` |

## 10. Gotchas

Inherited, plus: a human who sets a PBI to **Done by hand** before the PR merges lies about the
code being on `<DEFAULT_BRANCH>`; the dispatcher's stage-integrity check treats a Done PBI still
carrying a `stage:*` tag as a multi-state item and parks it.
