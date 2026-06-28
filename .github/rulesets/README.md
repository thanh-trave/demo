# Branch rulesets

GitHub does not read rulesets from the repo; these JSON files are the source of truth
kept in version control, and they must be applied to the repo settings manually (via
`gh api` or Settings -> Rules -> Rulesets -> New ruleset -> Import a ruleset).

## Design: minimal — enforce linear history, nothing else

This repo is the testbed for the gitflow slash-command workflow (`/gitflow:*`). The one
invariant the rulesets enforce is **linear history**: `main` and `staging` are a single
forward-only line, releases are fast-forwards, and there are no merge commits. The
rulesets are deliberately kept to only the rules that serve that invariant — no
require-a-PR rule, no required status checks. Either of those would block the direct
fast-forward push to `main` (`/gitflow:release`) and the force-with-lease push to
`staging` (`/gitflow:repair-staging`) for everyone without bypass rights, which fights the
Phase 1 "everyone runs the commands locally" design.

## Rule-by-rule

The `.json` files are consumed by `gh api --input`, which requires strict JSON, so they
can't carry inline comments. The one-sentence explanation of every rule lives here instead.

### main.json

- `deletion` - blocks deleting the `main` branch (trunk-protection safety net, not a linear-history rule, but cheap to keep).
- `required_linear_history` - rejects any push that would add a merge commit, so main stays one straight line.
- `non_fast_forward` - blocks force-pushes and history rewrites on main, while still allowing the fast-forward release push.
- `bypass_actors` (admin, `actor_id: 5`) - lets repo admins override the rules in an emergency; the normal flow never needs it.

### staging.json

- `deletion` - blocks deleting the `staging` branch (trunk-protection safety net).
- `required_linear_history` - rejects merge commits on staging so it stays linear (squash-from-feature and rebase-from-repair both qualify).
- (no `non_fast_forward`) - intentionally omitted so `/gitflow:repair-staging` can rebase staging onto main and force-push it (`--force-with-lease` is the safety net); the result is still linear.
- `bypass_actors` (admin, `actor_id: 5`) - emergency override only.

With this minimal set no bypass is needed for the day-to-day flow - the ff push and the
force-push both satisfy the rules. The admin bypass actor is kept only as an escape hatch.

## Differences from other setups

**vs. the live demo rulesets (what's currently applied on GitHub):** these files are
trimmed down. Compared to live, they DROP, on `main`: the `pull_request` rule (1 approval,
squash-only merge method) and the `required_status_checks` rule (`guard-cross-merge`); and
on `staging`: `required_status_checks` (`guard-cross-merge`). The retained rules
(`deletion`, `required_linear_history`, and `non_fast_forward` on main) are identical to
live. Until the live rulesets are re-applied from these files, GitHub still enforces the
richer live set.

**vs. trave-platform's committed rulesets:** trave keeps linear history by convention +
squash-only merge settings, not by a ruleset rule, so its `main` ruleset is
`deletion` + `non_fast_forward` and its `staging` ruleset is `deletion` only - neither has
`required_linear_history`. So relative to trave these files ADD `required_linear_history`
to both branches (enforcing the invariant at the ruleset level); `deletion` and
`non_fast_forward` on main match. Net: trave's `main` and ours share `deletion` +
`non_fast_forward`; trave's `staging` and ours share `deletion`.

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
