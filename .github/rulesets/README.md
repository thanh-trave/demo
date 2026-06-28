# Branch rulesets

GitHub does not read rulesets from the repo; these JSON files are the source of truth
kept in version control, and they must be applied to the repo settings manually (via
`gh api` or Settings -> Rules -> Rulesets -> New ruleset -> Import a ruleset).

## Design: minimal - enforce linear history, nothing else

This repo is the testbed for the gitflow slash-command workflow (`/gitflow:*`). The one
invariant the rulesets enforce is **linear history**: `main` and `staging` are a single
forward-only line, releases are fast-forwards, and there are no merge commits. The rulesets
are deliberately kept to only the rules that serve that invariant - no require-a-PR rule, no
required status checks. Either of those would block the direct fast-forward push to `main`
(`/gitflow:release`) and the force-with-lease push to `staging` (`/gitflow:repair-staging`)
for everyone without bypass rights, which fights the Phase 1 "everyone runs the commands
locally" design.

There are no bypass actors, and none are needed: the fast-forward release push and the
force-with-lease repair push both satisfy `required_linear_history` and `non_fast_forward`
on their own.

## Rule-by-rule

The `.json` files are consumed by `gh api --input`, which requires strict JSON, so they
can't carry inline comments. The one-sentence explanation of every rule lives here instead.

### main.json

- `deletion` - blocks deleting the `main` branch (trunk-protection safety net).
- `required_linear_history` - rejects any push that would add a merge commit, so main stays one straight line.
- `non_fast_forward` - blocks force-pushes and history rewrites on main, while still allowing the fast-forward release push.
- `bypass_actors: []` - empty; no bypass is needed because the ff release push already satisfies the rules above.

### staging.json

- `deletion` - blocks deleting the `staging` branch (trunk-protection safety net).
- `required_linear_history` - rejects merge commits on staging so it stays linear (squash-from-feature and rebase-from-repair both qualify).
- (no `non_fast_forward`) - intentionally omitted so `/gitflow:repair-staging` can rebase staging onto main and force-push it (`--force-with-lease` is the safety net); the result is still linear.
- `bypass_actors: []` - empty; no bypass is needed.

## Effect on the slash commands

With this minimal set no bypass is needed for the day-to-day flow - every command's push
satisfies the rules:

- `/gitflow:ship-to-staging` lands a feature through a squash PR to `staging`.
- `/gitflow:hotfix` lands an urgent fix through a squash PR to `main`.
- `/gitflow:release` fast-forwards `main` with a direct push - allowed (a ff is not a
  force-push and adds no merge commit).
- `/gitflow:repair-staging` force-pushes `staging` with lease - allowed (staging has no
  `non_fast_forward` rule).

If you later want to require a reviewed PR before anything lands, add a `pull_request` rule
back to one or both files - but then also add a `bypass_actors` entry (admin role
`{ "actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always" }`, or a release
bot) so `release`/`repair-staging` can still push directly, and re-apply.

## Differences from other setups

**vs. trave-platform:** trave's committed rulesets keep linear history by convention +
squash-only merge settings, so its `main` is `deletion` + `non_fast_forward` and its
`staging` is `deletion` only. Trave's *live* `PR Force` ruleset adds `deletion` +
`non_fast_forward` + `pull_request` on the default branch (`main`) only - and it is
currently **disabled**, never enforced, and never applied to `staging`. So relative to
trave, these files add `required_linear_history` on **both** branches (enforcing the
invariant at the ruleset level); `deletion` and `non_fast_forward` on main match.

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
