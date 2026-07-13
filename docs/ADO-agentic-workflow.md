# SDLC Revision Proposal: Feature-Dominant Orchestration

## Summary

Collapse the current dual-mode pipeline (Feature-orchestrated `delivery.*` vs standalone `stage.*`)
into a single model: **every unit of deliverable work is a Feature**. This matches the human-run
SDLC, where the Feature is already the home of the BRD and acceptance criteria. PBIs lose all
lifecycle state and become pure repo-scoped execution records. PSIs get their own triage
lifecycle as problem *reports*, distinct from work, and convert into Features only when
diagnosis determines a code change is required.

The migration is subtractive: the standalone `stage.*` pipeline, its tags, its human gates, and
its dispatcher branch are deleted rather than replaced.

## ADO Hierarchy (unchanged, clarified)

```
Epic → Feature → PBI → Task | PSI
```

- **Epic** — business context only; never enters the pipeline.
- **Feature** — the sole orchestration state machine. Owns BRD, acceptance criteria, delivery
  state (`delivery.*`), dependency ordering, and both human gates.
- **PBI** — repo-scoped (and sprint-scoped for capacity reporting) block of work under exactly
  one Feature. Carries **no lifecycle state**: only a repo routing tag (`repo.core-api` /
  `repo.webapp` / `repo.utility`) and `result:*` evidence tags posted by workers.
- **Task** — implementation substep created and closed under a PBI by the build worker.
- **PSI** — sibling of Task; a production support incident report. Lives under a PBI, initially
  the Production Support stub structure (see PSI Lifecycle below).

## 1. Everything Is a Feature

There is no standalone execution mode. All planned work, all bug fixes, all code changes arising
from PSIs enter as (or under) a Feature. Rationale:

- The Feature is where the durable business record lives (BRD, AC). Orchestrating anything else
  means orchestrating an item with no requirements contract.
- One state machine → one tag vocabulary, one dispatcher walk, one prioritization surface for
  the human gates.
- Matches the human-run process, so the agentic pipeline and the human pipeline stop diverging.

The Feature pipeline is unchanged:

```
delivery.intake → [delivery.design] → delivery.queued → delivery.build → delivery.verify
  → delivery.audit → delivery.ready → delivery.shipping → delivery.complete
```

Two human gates remain: **queued → build** (prioritization) and **ready → shipping**
(ship approval). Design gate (`design.visual-required` / `design.approved`) is unchanged.

Small work is not exempted: a one-repo tweak is a lightweight Feature with one PBI. The cost of
a thin Feature is lower than the cost of maintaining a second pipeline. For genuinely urgent
production fixes, see the Hotfix Profile under PSI Lifecycle.

### Within-Feature progression (kept from current design)

The Feature owns the *gates*; children do not march in lockstep. The dependency-aware barrier is
retained: providers and independent PBIs build first, consumers start once every predecessor has
posted a contract-ready comment (contract shape, compatibility/nullability expectations,
migration notes, commit/PR reference). Consumers do not wait for providers to *ship*.

## 2. Stateless PBIs

PBIs carry no `stage.*`, no `delivery.*`, no gates, no independent progression. Their entire
machine-readable surface:

| Tag | Meaning |
|---|---|
| `repo.core-api` \| `repo.webapp` \| `repo.utility` | exactly one; selects worktree, repo instructions, build/test/verification behavior |
| `result:<phase>-passed` / `result:<phase>-blocked` | evidence posted by lane workers, consumed by the Feature's phase advancement |

Consequences:

- The rule "a Feature child must never receive `stage.*`" becomes structurally impossible to
  violate instead of merely forbidden — there is no state to misapply.
- The dispatcher only ever walks Feature phases. The standalone lane-processing branch is
  deleted.
- **PBIs are created only for repos that have work.** Intake creates one PBI per affected repo
  (typically front-end + back-end; utility when needed) — never speculatively. An absent PBI is a
  trivially satisfied dependency at the contract barrier.
- Sprint assignment on PBIs is reporting metadata only, never a pipeline input. Features span
  sprints freely; iteration boundaries never force Feature splits.

PBI content requirements are unchanged: structured description, independently verifiable
acceptance criteria derived from the Feature's, contract boundary, affected surfaces, test plan.

### Naming (unchanged)

Feature-child branches: `feature/<FEATURE-ID>-<feature-slug>/<PBI-ID>-<repo>`, coordinated
across repos. Hotfix branches: `fix/<PSI-ID>-<slug>` (see Hotfix Profile).

## 3. PSI Lifecycle

A PSI is a **report, not work**. It never enters the delivery pipeline. It runs its own triage
lifecycle and, when code change is required, *spawns* a Feature and is re-homed under it.

### Structural home

Stub PSIs are consumed under the standing Production Support structure:

```
Production Support (Epic)
  → Production Support (Feature)          ← never enters delivery.*; container only
    → Prod Support {Sprint} (PBI)         ← stateless, like all PBIs
      → PSI                               ← carries psi.* lifecycle tags
```

The Production Support Feature is explicitly excluded from dispatcher Feature-phase processing
(it is a container, not a deliverable). The dispatcher instead processes PSIs directly by their
`psi.*` state — this is the one non-Feature walk the dispatcher retains, and it is
investigation-lane work, not build work.

### PSI states

```
psi.reported → psi.triage → psi.diagnosed → psi.decided → psi.pending-fix → psi.resolved
                                          ↘ (closed at decision for non-code outcomes)
```

| State | Owner | Exit criteria |
|---|---|---|
| `psi.reported` | dispatcher | picked up for triage |
| `psi.triage` | investigation worker | severity assigned; repro attempted; affected repo(s)/surface identified |
| `psi.diagnosed` | investigation worker | root cause (or best hypothesis) documented on the PSI: repro steps, logs, suspected component, config vs code determination |
| `psi.decided` | **human gate** | one of the four decision outcomes recorded |
| `psi.pending-fix` | automatic | linked Feature reaches `delivery.complete` **and** fix verification passes |
| `psi.resolved` | terminal | — |

The investigation worker is a distinct lane prompt (repro, log-digging, config inspection,
read-only against the repos) — not the build worker. It never creates branches or PRs.

### Decision outcomes (human gate at `psi.decided`)

1. **No change** — working as designed, user error, environment/config issue resolved
  operationally. PSI closed with rationale.
2. **Documentation only** — wiki/docs updated (may be done by the investigation worker); PSI
  closed with a link to the change.
3. **Duplicate** — linked to the existing PSI or Feature; closed.
4. **Needs code** — a new Feature (and its repo PBIs) is created with the PSI's diagnosis as the
  intake payload; **the PSI is re-homed** out of the Production Support stub structure to sit
  under the new Feature's relevant PBI. The PSI moves to `psi.pending-fix`.

### Re-homing and closure

On the needs-code outcome:

- New Feature's BRD/AC are seeded from the PSI diagnosis (repro, root cause, expected vs actual
  behavior). The AC must include the PSI's original repro as a verification case.
- PSI is re-parented under the new Feature's PBI (the repo where the fix lands; if multi-repo,
  the primary/root-cause repo), preserving the report's history in situ.
- The PSI and Feature are linked both directions (ADO related links + tag `psi.origin:<PSI-ID>`
  on the Feature for machine traversal).
- When the Feature reaches `delivery.complete`, PSI resolution requires **verification against
  the original report's repro**, not merely the Feature's AC — this closes the loop between
  "we shipped a fix" and "the reported problem is gone." On pass, `psi.resolved`; on fail, the
  PSI bounces the Feature back with the failing repro (Feature re-enters build).

### Hotfix Profile (expedited Feature)

Sev-1/critical PSIs cannot wait for full intake → design → queued. The needs-code outcome may
mark the spawned Feature `delivery.hotfix`, which:

- Skips the design gate entirely.
- Pre-approves the queued gate (the human decision at `psi.decided` *is* the prioritization
  decision — one human touch, not two).
- Retains verify and audit (audit especially: emergency changes are where security regressions
  hide), but runs them concurrently rather than serially.
- Retains the ready → shipping human gate. Ship approval is never skipped.

Make this profile explicit and cheap, or engineers will bypass the system under incident
pressure and the bypass becomes the real process.

### Bugs found during a Feature vs escaped defects

- Defects found **while a Feature is in flight** are additional child work under that same
  Feature (new Task or PBI behind the same gates) — not PSIs, not new Features.
- **Escaped/production** defects arrive as PSIs and follow the lifecycle above. There is no
  third path.

## 4. Tag Vocabulary After Migration

| Set | Applies to | Values |
|---|---|---|
| `delivery.*` | Feature | intake, design, queued, build, verify, audit, ready, shipping, complete, hotfix |
| `psi.*` | PSI | reported, triage, diagnosed, decided, pending-fix, resolved |
| `repo.*` | PBI | core-api, webapp, utility |
| `result:*` | PBI | `<phase>-passed`, `<phase>-blocked` |
| `psi.origin:*` | Feature | back-link to spawning PSI |
| `sdlc.*` | operational | pr-open, hold, needs-human, dispatch-running, wip, shipped (unchanged) |

**Deleted:** the entire `stage.*` set and every prompt/parser branch that reads it.

## 5. Dispatcher Changes

- Remove the standalone-lane processing branch; the cycle walks Feature phases (priority order
  shipping → audit → verify → hold → build → design → intake, unchanged) plus a PSI triage pass.
- PSI triage pass: claim `psi.reported`/`psi.triage` items for the investigation worker; surface
  `psi.diagnosed` items to the human decision queue alongside `delivery.queued` Features — one
  prioritization surface for the human.
- Worker CLAIM → WORK → EMIT → STOP contract unchanged; the investigation worker is a new lane
  prompt with read-only repo access.
- Helper script: remove `stage.*` handling; add `psi.*` transitions and the re-homing operation
  (re-parent + dual link + `psi.origin` tag) as a single atomic helper command — re-homing by
  hand is the likeliest place for the hierarchy to rot.
- Lock hygiene (independent of this proposal but worth bundling): automate the stale-lock
  reaper on a schedule and prefer a heartbeat-renewed lease over the fixed two-hour age
  heuristic — a slow-but-alive worker and a dead one currently look identical.

## 6. Migration Steps

1. Freeze creation of new standalone PBIs/Bugs (announce; helper script rejects `stage.*` on
   create).
2. Drain in-flight standalone items to terminal states under the old rules.
3. Wrap any remaining planned standalone work in thin Features.
4. Stand up the Production Support container structure and PSI states; convert existing stub
   PSIs in place (they are already in the right hierarchy).
5. Delete `stage.*` handling from dispatcher, helper script, lane prompts, and wiki docs.
6. Add investigation-lane prompt + hotfix profile.
7. Update SDLC-PIPELINE.md / ADO-HIERARCHY.md to this model.

## 7. What Is Deliberately Kept

- Two human gates on Features (queued, shipping) — no more, no fewer.
- Design gate and storyboard requirements for material webapp changes.
- Contract-ready barrier and provider/consumer ordering across repos.
- Verify's branch-checkout/full-suite behavior; audit's OWASP/architecture scope.
- CLAIM/WORK/EMIT/STOP worker contract, idempotency rules, singleton dispatch lock.
- Branch/PR governance (everything targets development; main is human-only).
- The helper script as the mandatory tag mutation path (ADO `--fields System.Tags` append
  behavior makes raw updates unsafe).
