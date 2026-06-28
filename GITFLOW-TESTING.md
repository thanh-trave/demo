# Testing the gitflow slash-command workflow

This repo is a testbed for the linear-history git workflow described in the Notion doc
"Git workflow: Linear history - with Claude slash commands". This file is the runbook:
the one-time setup, how to import the rulesets, and a concrete scenario per slash command
so you can exercise every case end to end.

The five commands under test (in `.claude/commands/gitflow/`):

| Command | Use when | Effect |
| --- | --- | --- |
| `/gitflow:ship-to-staging` | a feature/fix is ready | squash-merge your branch -> `staging` |
| `/gitflow:release` | weekly release (or partial via `staging-2`) | fast-forward `main` -> `staging` + tag |
| `/gitflow:hotfix` | urgent prod bug | squash a fix -> `main`, tag, then repair staging |
| `/gitflow:repair-staging` | after a hotfix / when release reports main diverged | rebase `staging` onto `main`, force-push |
| `/gitflow:sync-branch` | your feature branch was cut before staging was rebased | rebase your branch onto the new `staging` |

---

## 0. Prerequisites

- `gh` CLI authenticated (`gh auth status`) with access to `thanh-trave/demo`.
- You are running the slash commands from inside a clone of this repo.
- The commands do all the git mutations themselves (fetch, rebase, push, tag) with a
  confirmation before each push. The setup steps below only build the branch/commit state
  that each command then acts on.

---

## 1. One-time repo settings

Most of this is already configured on `thanh-trave/demo`. Verify or re-apply:

### 1a. Merge settings (squash-only) - already set

The squash commit must be `<PR title> (#n)` + PR description, which the commands rely on:

```bash
gh api repos/thanh-trave/demo --method PATCH \
  -F allow_squash_merge=true \
  -F allow_merge_commit=false \
  -F allow_rebase_merge=false \
  -F squash_merge_commit_title=PR_TITLE \
  -F squash_merge_commit_message=PR_BODY
```

### 1b. `GUARD_PAT` secret - already set

`guard-cross-merge.yml` re-drafts any staging<->main PR. The default `GITHUB_TOKEN` can't
run `convertPullRequestToDraft`, so the workflow uses a `GUARD_PAT` secret: a fine-grained
PAT with "Pull requests: Read and write" on this repo. Confirm it exists:

```bash
gh secret list   # expect GUARD_PAT in the list
```

### 1c. Import the rulesets

The source of truth is `.github/rulesets/main.json` and `staging.json` (see that folder's
README for the rule-by-rule rationale). GitHub does not read them from the repo - import
them manually.

Two existing rulesets are already applied to this repo (the richer "live" set):
`Rule main` (id `17898143`) and `Rule staging` (id `17898719`). To replace them with the
minimal versions in these files, **update in place** with PUT:

```bash
gh api repos/thanh-trave/demo/rulesets/17898143 --method PUT --input .github/rulesets/main.json
gh api repos/thanh-trave/demo/rulesets/17898719 --method PUT --input .github/rulesets/staging.json
```

Or start fresh - delete the two existing ones and POST new:

```bash
gh api repos/thanh-trave/demo/rulesets/17898143 --method DELETE
gh api repos/thanh-trave/demo/rulesets/17898719 --method DELETE
gh api repos/thanh-trave/demo/rulesets --method POST --input .github/rulesets/main.json
gh api repos/thanh-trave/demo/rulesets --method POST --input .github/rulesets/staging.json
```

(UI alternative: Settings -> Rules -> Rulesets -> New ruleset -> Import a ruleset.)

With the minimal rulesets, the `/gitflow:release` ff-push and `/gitflow:repair-staging`
force-push work without admin bypass. If you leave the richer live rulesets in place
instead, you'll need the admin bypass (you have it) for those two pushes.

---

## 2. Clean baseline before testing

Start every session from "main and staging are the same commit". This resets shared
branches, so only do it on the testbed:

```bash
git fetch origin --prune
git checkout main && git reset --hard origin/main
git checkout staging && git reset --hard main && git push --force-with-lease origin staging
```

Now `main == staging`. Delete leftover test branches if present:

```bash
git branch -D feat-login hot-fix-crash feat-inflight staging-2 2>/dev/null
git push origin --delete feat-login hot-fix-crash feat-inflight staging-2 2>/dev/null
```

---

## 3. Test case 1 - Feature -> staging (`/gitflow:ship-to-staging`)

**Proves:** a feature branch lands on staging as a single squash commit via a PR, staging
stays linear, the branch is kept.

**Setup** - make a feature branch with a couple of commits:

```bash
git checkout main && git pull --ff-only
git checkout -b feat-login
printf 'login\n' > login.txt && git add login.txt && git commit -m "add login form"
printf 'login validation\n' >> login.txt && git commit -am "validate login"
git push -u origin feat-login
```

(To also test Linear-id title formatting, name the branch `feat-TRA-101-login` - the
generated PR title becomes `TRA-101: <title>`.)

**Run:** stand on `feat-login`, run `/gitflow:ship-to-staging`. Confirm the prompts: it
rebases onto `origin/staging`, proposes a PR title (OK it), opens the PR to `staging`, and
on your say-so squash-merges it.

**Verify:**

```bash
git fetch origin
git log --oneline origin/staging | head -3   # one new commit: "add login form (#N)" style
git branch -a | grep feat-login              # branch still exists (not deleted)
git merge-base --is-ancestor origin/main origin/staging && echo "staging is linear ahead of main"
```

---

## 4. Test case 2 - Weekly release (`/gitflow:release`)

**Proves:** `main` fast-forwards to `staging` (identical SHAs), a tag is pushed, the
staging->main review PR is auto-drafted by the guard and auto-closes on the ff push.

**Setup** - staging must be ahead of main (do Test 1 first). Open the review PR (the
command can also create it):

```bash
gh pr create --base main --head staging \
  --title "Release - $(date +%Y-%m-%d)" --body "Weekly release"
```

Watch it get force-drafted by `guard-cross-merge` within a few seconds (a red X check plus
a comment pointing you to `/gitflow:release`). That is expected - the release does NOT go
through the merge button.

**Run:** `/gitflow:release`. Confirm: team-notified prompt, it checks the PR's CI finished,
verifies `origin/main` is an ancestor of `origin/staging`, shows the shipping commits,
proposes a tag like `v1.0.0-YYYYMMDD` (confirm or override), then ff-pushes `main` + tag.

**Verify:**

```bash
git fetch origin --tags
[ "$(git rev-parse origin/main)" = "$(git rev-parse origin/staging)" ] && echo "main == staging"
git tag --sort=-creatordate | head -1        # the new release tag
gh pr list --base main --state all | head    # the review PR shows merged/closed
```

---

## 5. Test case 3 - Hotfix (`/gitflow:hotfix`)

**Proves:** an urgent fix squashes onto `main`, gets a patch-bumped tag, and staging is
auto-repaired (rebased onto main) so the next release can still ff.

**Setup** - first put some unreleased work on staging so the repair step has something to
replay, then cut a hotfix from main:

```bash
# unreleased staging work (ship a quick feature via Test 1's flow, or fast version:)
git checkout staging && git pull --ff-only
git checkout -b feat-extra && printf 'x\n' > extra.txt && git add extra.txt && git commit -m "add extra"
git push -u origin feat-extra
#   -> run /gitflow:ship-to-staging on feat-extra so staging is ahead of main

# the hotfix branch, cut from main:
git checkout main && git pull --ff-only
git checkout -b hot-fix-crash
printf 'fix\n' > hotfix.txt && git add hotfix.txt && git commit -m "fix startup crash"
git push -u origin hot-fix-crash
```

**Run:** stand on `hot-fix-crash`, run `/gitflow:hotfix`. It requires a `hot-fix-*`->main
PR (creates it), squash-merges with a forced `hotfix:` subject, tags `vX.Y.(Z+1)-date`,
pushes main, then runs the staging repair as part of the hotfix.

**Verify:**

```bash
git fetch origin --tags
git log --oneline origin/main | head -2                 # top commit is "hotfix: ... (#N)"
git tag --sort=-v:refname | head -1                     # patch bump, e.g. v1.0.1-YYYYMMDD
git merge-base --is-ancestor origin/main origin/staging && echo "staging repaired (main is ancestor)"
```

---

## 6. Test case 4 - Repair staging (`/gitflow:repair-staging`)

**Proves:** when `main` has moved ahead of `staging`, staging's unreleased work is rebased
on top of main and force-pushed, staying linear with no duplicated commits.

**Setup** - manufacture a divergence (main has a commit staging lacks, and staging has its
own unreleased commit):

```bash
git checkout main && git pull --ff-only
printf 'm\n' > main-only.txt && git add main-only.txt && git commit -m "main moved ahead"
git push origin main                       # ff + linear, allowed
git checkout staging && git pull --ff-only
printf 's\n' > staging-only.txt && git add staging-only.txt && git commit -m "staging work"
git push --force-with-lease origin staging
git merge-base --is-ancestor origin/main origin/staging || echo "diverged (expected)"
```

**Run:** `/gitflow:repair-staging`. It records the old staging tip (note it - needed for
Test 5), makes a backup ref, rebases staging onto main, shows a range-diff so you see
nothing was lost, then force-pushes with lease.

**Verify:**

```bash
git fetch origin
git merge-base --is-ancestor origin/main origin/staging && echo "repaired"
git log --oneline origin/main..origin/staging   # only "staging work", replayed on top of main
```

---

## 7. Test case 5 - Sync an in-flight branch (`/gitflow:sync-branch`)

**Proves:** a feature branch cut from the OLD staging is cleanly rebased onto the NEW
(rebased) staging, dropping old-staging twins and replaying only your commits.

**Setup** - cut a branch from staging, capture the old tip, THEN rebase staging (run
Test 3 or Test 4 in between so staging is force-pushed with new SHAs):

```bash
git checkout staging && git pull --ff-only
git checkout -b feat-inflight
printf 'i\n' > inflight.txt && git add inflight.txt && git commit -m "inflight work"
git push -u origin feat-inflight
OLD_TIP=$(git rev-parse origin/staging) && echo "OLD_TIP=$OLD_TIP"
#   -> now run Test 3 (hotfix) or Test 4 (repair) so staging gets rebased + force-pushed
```

**Run:** stand on `feat-inflight`, run `/gitflow:sync-branch` (or pass the tip explicitly
if auto-detection struggles: `/gitflow:sync-branch <OLD_TIP>`). It backs up the branch,
rebases it onto the new staging, and asks before force-pushing the branch.

**Verify:**

```bash
git log --oneline -5                            # only "inflight work" sits on top of staging
git merge-base --is-ancestor origin/staging HEAD && echo "branch is on current staging"
```

---

## 8. Test case 6 - Partial release (`/gitflow:release staging-2`)

**Proves:** a subset of staging ships while holding a commit back, via a disposable
`staging-2`, then the original staging is reconciled onto main.

**Setup** - staging has three commits; hold the middle one back. Build `staging-2` from
main and cherry-pick only the ones to ship:

```bash
git checkout staging && git pull --ff-only
# assume staging has S1, S2 (hold), S3 on top of main - note their SHAs:
git log --oneline origin/main..origin/staging
git checkout -b staging-2 origin/main
git cherry-pick <S1> <S3>          # skip S2; resolve any conflict if S3 depended on S2
git push -u origin staging-2
```

**Run:** `/gitflow:release staging-2`. It requires a `staging-2`->main review PR (drafted
by the guard), fast-forwards main to staging-2, tags, then reconciles the original staging
onto main (the repair operation) and deletes `staging-2`.

**Verify:**

```bash
git fetch origin --prune --tags
git log --oneline origin/main | head -3                  # S1', S3' on main
git merge-base --is-ancestor origin/main origin/staging && echo "staging reconciled"
git log --oneline origin/main..origin/staging            # S2' (the held commit) replayed on top
git branch -a | grep staging-2 || echo "staging-2 deleted"
```

---

## 9. Test case 7 - Guard blocks staging<->main UI merges (`guard-cross-merge.yml`)

**Proves:** the workflow keeps both cross-merge directions out of the merge button,
independent of the rulesets.

**Run + verify (staging -> main):**

```bash
gh pr create --base main --head staging --title "guard test" --body "should be drafted"
sleep 10
gh pr view <n> --json isDraft,statusCheckRollup   # isDraft: true; guard-cross-merge failed
gh pr ready <n>                                    # try to undraft...
sleep 10
gh pr view <n> --json isDraft -q .isDraft          # back to true - re-drafted
gh pr close <n>
```

**Reverse direction (main -> staging)** gets the repair-staging message:

```bash
gh pr create --base staging --head main --title "guard test reverse" --body "should be drafted"
# ...verify drafted, then close
```

---

## 10. Reset and re-run

After any test, return to the clean baseline (Section 2) before the next one. The
`backup/*` refs created by repair/sync can be deleted once you've confirmed the result:

```bash
git branch | grep backup/ | xargs -r git branch -D
```
