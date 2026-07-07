# Label taxonomy

The pipeline is driven entirely by GitHub labels. Create these once per repo.

## The labels

| Label | Colour | Meaning |
|---|---|---|
| `stage:intake` | `#0e8a16` | Awaiting triage/routing. |
| `stage:queued` | `#fbca04` | Triaged, ready to build — **the human throttle**. No worker. |
| `stage:build` | `#1d76db` | Being implemented (or waiting to be). |
| `stage:verify` | `#5319e7` | Awaiting full-suite + real-run validation. |
| `stage:audit` | `#b60205` | Awaiting read-only security/invariant review. |
| `stage:ship` | `#0052cc` | Awaiting docs fan-out + PR. |
| `stage:design` | `#c5def5` | *(optional)* Awaiting design/storyboard. Omit if folding design into intake. |
| `sdlc:wip` | `#d93f0b` | Per-issue lock. Machine-owned, volatile. Paired with an `sdlc:claim` comment. |
| `sdlc:needs-human` | `#e99695` | Parked — a worker needs a human decision. Automation never advances it. |
| `sdlc:hold` | `#000000` | Human keep-off. No worker touches it. Also marks the dispatcher-lock issue. |
| `priority:critical` | `#b60205` | Claimed first within a lane. |
| `priority:medium` | `#fbca04` | Default priority. |
| `priority:future` | `#c2e0c6` | Claimed last. |
| `blocked` | `#d876e3` | *(optional)* Item bounced to queued as not-yet-buildable; readiness axis. |
| `ready` | `#0e8a16` | *(optional)* Blocker cleared; complements `blocked`. |

## Create them (`gh`)

Run from the repo, authenticated with `repo` scope. `--force` makes it idempotent (updates colour if
the label already exists).

```bash
# stages
gh label create "stage:intake"  --color 0e8a16 --description "Awaiting triage/routing" --force
gh label create "stage:queued"  --color fbca04 --description "Ready to build — human throttle (no worker)" --force
gh label create "stage:build"   --color 1d76db --description "Being implemented" --force
gh label create "stage:verify"  --color 5319e7 --description "Awaiting validation" --force
gh label create "stage:audit"   --color b60205 --description "Awaiting security/invariant review" --force
gh label create "stage:ship"    --color 0052cc --description "Awaiting docs fan-out + PR" --force
# optional design lane
gh label create "stage:design"  --color c5def5 --description "Awaiting design/storyboard (optional)" --force

# sdlc control
gh label create "sdlc:wip"          --color d93f0b --description "Per-issue lock (machine-owned)" --force
gh label create "sdlc:needs-human"  --color e99695 --description "Parked — needs a human decision" --force
gh label create "sdlc:hold"         --color 000000 --description "Human keep-off — no worker touches it" --force

# priority
gh label create "priority:critical" --color b60205 --description "Claimed first within a lane" --force
gh label create "priority:medium"   --color fbca04 --description "Default priority" --force
gh label create "priority:future"   --color c2e0c6 --description "Claimed last" --force

# optional readiness axis
gh label create "blocked" --color d876e3 --description "Not yet buildable (bounced to queued)" --force
gh label create "ready"   --color 0e8a16 --description "Blocker cleared" --force
```

## Per-issue variant only: the dispatcher mutex

The concurrent variant needs a singleton-lock issue. Create it once and pin it:

```bash
n=$(gh issue create --title "sdlc:dispatch-lock" \
  --body "Dispatcher mutex. The dispatcher posts lock/unlock comments here; no worker touches it." \
  --label "sdlc:hold" --json number --jq .number)
gh issue pin "$n"
```
