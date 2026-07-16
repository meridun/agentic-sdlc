# Repo conventions

## Git workflow (this repo itself)

This repo eats its own dog food: the flow is **`feature → dev → master`**, i.e. the template's
`{feature} → <DEFAULT_BRANCH> → <PROD_BRANCH>` with `<DEFAULT_BRANCH> = dev` and
`<PROD_BRANCH> = master`.

- Cut feature branches (`<type>/<slug>` or `<type>/<issue#>-<slug>`) from `dev`.
- **PRs target `dev`, not `master`** — even docs-only changes.
- `master` moves only by a human-initiated `dev → master` promotion PR.
- After a promotion merges, merge `master` back into `dev` so the branches stay even.
- Push all commits for a PR **before** it is merged; a commit pushed after the merge silently
  misses the train and needs a follow-up PR.

## What lives where

- `docs/` — the *why*: model, invariants, composability spec, adoption guide.
- `prompts/sdlc/` — executable worker prompts; `README.md` there is the binding universal
  worker loop, lane files define only WORK/EMIT specifics.
- Keep the two in sync: a rule stated in `docs/AgenticSDLC.md`'s invariants must have its
  operational counterpart in `prompts/sdlc/README.md` and any affected lane files.
