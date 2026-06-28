# Branch rulesets

GitHub does not read rulesets from the repo; these JSON files are the source of truth
kept in version control, and they must be applied to the repo settings manually (via
`gh api` or Settings -> Rules -> Rulesets -> New ruleset -> Import a ruleset).

## Design: enforce linear history AND require a reviewed PR

This repo is the testbed for the gitflow slash-command workflow (`/gitflow:*`). The
rulesets enforce two things: **linear history** (`main` and `staging` are a single
forward-only line, no merge commits) and **a pull request before anything lands** on
either branch (squash-only merge method). `main` requires 1 approving review; `staging`
requires none, so a staging PR can be self-merged without waiting on a reviewer.

There are no bypass actors, so the rules apply to everyone, including admins. That makes
this the *enforced* posture (closer to the Notion doc's Phase 2 than the loose Phase 1):
because `pull_request` forbids direct pushes, the two commands that push directly -
`/gitflow:release` (fast-forward push to `main`) and `/gitflow:repair-staging`
(force-with-lease push to `staging`) - are blocked as written. To run those locally you
must add a bypass actor (an admin role, or a release bot for real Phase 2); without one,
`main` only advances through an approved squash PR. See "Effect on the slash commands".

## Rule-by-rule

The `.json` files are consumed by `gh api --input`, which requires strict JSON, so they
can't carry inline comments. The one-sentence explanation of every rule lives here instead.

### main.json

- `deletion` - blocks deleting the `main` branch (trunk-protection safety net).
- `required_linear_history` - rejects any push that would add a merge commit, so main stays one straight line.
- `non_fast_forward` - blocks force-pushes and history rewrites on main, while still allowing a fast-forward update.
- `pull_request` (1 approval, `allowed_merge_methods: [squash]`) - changes must arrive through a PR with at least one approving review, merged by squash; no direct pushes.
- `bypass_actors: []` - empty, so nobody bypasses; the rules above bind admins too.

### staging.json

- `deletion` - blocks deleting the `staging` branch (trunk-protection safety net).
- `required_linear_history` - rejects merge commits on staging so it stays linear (squash-from-feature and rebase-from-repair both qualify).
- `pull_request` (0 approvals, `allowed_merge_methods: [squash]`) - requires a squash PR but no approving review, so a staging PR can be self-merged; still blocks direct pushes.
- (no `non_fast_forward`) - omitted so a rebased staging *could* be force-pushed, but note the `pull_request` rule still blocks direct pushes without a bypass actor.
- `bypass_actors: []` - empty; no one bypasses.

## Effect on the slash commands

With `pull_request` active and no bypass actor, the direct-push commands cannot push:

- `/gitflow:release` fast-forwards `main` with a direct push - **blocked**; `main` would
  instead have to advance by merging the staging->main PR, which contradicts the
  fast-forward-only design.
- `/gitflow:repair-staging` force-pushes `staging` - **blocked** for the same reason.
- `/gitflow:ship-to-staging` lands through a squash PR to `staging`, which needs no
  approval, so it merges without waiting on a reviewer. `/gitflow:hotfix` targets `main`,
  so its squash PR still needs 1 approval.

To keep `release`/`repair-staging` working, add a `bypass_actors` entry (admin role
`{ "actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always" }`, or a release
bot) to both files and re-apply.

## Differences from other setups

**vs. trave-platform:** trave's committed rulesets keep linear history by convention +
squash-only merge settings, so its `main` is `deletion` + `non_fast_forward` and its
`staging` is `deletion` only. Trave's *live* `PR Force` ruleset adds `deletion` +
`non_fast_forward` + `pull_request` on the default branch (`main`) only - and it is
currently **disabled**, never enforced, and never applied to `staging`. So relative to
trave, these files enforce more: `required_linear_history` and `pull_request` on **both**
branches, actively.

## staging<->main UI merges are blocked separately

`.github/workflows/guard-cross-merge.yml` converts any staging<->main PR to **draft** (and
re-drafts it on every "Ready for review" click). Draft PRs have no merge button and
`gh pr merge` refuses them, so a UI merge commit between the two branches can't happen.
This is handled by the workflow, not a ruleset, so it doesn't need to be a required status
check. Normal PRs (`feature-*`/`fix-*` -> staging, `hot-fix-*` -> main) are untouched.

## Apply

```bash
gh api repos/thanh-trave/demo/rulesets \
  --method POST --input .github/rulesets/main.json
gh api repos/thanh-trave/demo/rulesets \
  --method POST --input .github/rulesets/staging.json
```

To update an existing ruleset, find its id (`gh api repos/thanh-trave/demo/rulesets`)
and `--method PUT` to `rulesets/<id>` with the same `--input`.

## Repo merge settings (one-off, not part of the rulesets)

Squash merge only, set in Settings -> General -> Pull Requests, or:

```bash
gh api repos/thanh-trave/demo --method PATCH \
  -F allow_squash_merge=true \
  -F allow_merge_commit=false \
  -F allow_rebase_merge=false \
  -F squash_merge_commit_title=PR_TITLE \
  -F squash_merge_commit_message=PR_BODY
```

(`PR_TITLE`/`PR_BODY` keeps the "squash commit = `<PR title> (#n)` + PR description"
convention the gitflow commands rely on.)

## guard-cross-merge secret

`guard-cross-merge.yml` needs a `GUARD_PAT` secret (a fine-grained PAT with
"Pull requests: Read and write" on this repo). The default `GITHUB_TOKEN` cannot run
`convertPullRequestToDraft`. The secret is already set on this repo.
