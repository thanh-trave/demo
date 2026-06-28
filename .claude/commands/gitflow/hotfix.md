---
description: Hotfix — squash an urgent fix into main, tag, and trigger the staging repair (Phase 1, local-driven)
allowed-tools: Bash(git:*), Bash(gh:*), Bash(cd:*), Bash(pwd:*)
---

# /gitflow:hotfix

Ship an urgent production fix straight to `main` as a single squash commit, tag it, then
reconcile staging so the linear-history invariant holds again. The user always has a
`hot-fix-*` branch already — this command ships it; it does not create the fix.

Optional argument — the hot-fix branch name (if you're standing on another branch), or the
number of its PR to `main`: $ARGUMENTS

## Preflight (ALWAYS)

1. `git rev-parse --show-toplevel`; if not a repo, stop. `cd` to root.
2. `git fetch origin --tags` so `origin/main` and `origin/staging` are current. Capture
   current branch + `git status --porcelain`.

## Resolve the hot-fix branch (the user always has one already)

3. Work out which branch we're shipping from $ARGUMENTS:
   - **A PR number** (e.g. `123`): treat it as the existing hot-fix → main PR. Read it
     (`gh pr view <n> --json headRefName,baseRefName,state`); its head branch is the hot-fix
     branch. Confirm the base is `main`; if the base is `staging`, STOP and flag it — a
     hotfix PR must target `main`.
   - **A branch name:** ship that branch (no need to be standing on it).
   - **Nothing given:** if the current branch is a `hot-fix-*` branch, confirm with the user
     it's the one to ship. Otherwise look for an open hot-fix → main PR
     (`gh pr list --base main --state open`); if exactly one obvious `hot-fix-*` candidate,
     confirm it. If still ambiguous or none found, **ask the user for the hot-fix branch
     name** — never guess.
     From here on `<branch>` = the resolved hot-fix branch and `<n>` = its PR number if known.

## Make sure the branch sits on main

4. A hotfix must reach production without dragging in unreleased staging work, so it has to
   sit on top of `main`. Check where it was cut from:
   `git merge-base --is-ancestor origin/main <branch>`.
   - **Already based on main (true):** nothing to rebase — go to step 5 (no rebase happened).
   - **Cut from staging (false):** it likely carries unreleased commits. Do NOT blindly
     rebase the whole branch onto main. First **show the user exactly what's on the branch**:
     `git log --oneline origin/main..<branch>` (everything not yet on main) and
     `git log --oneline origin/staging..<branch>` (what's unique past staging). Walk through
     these with the user and **get explicit confirmation of the precise set of commits that
     ARE the hot-fix** (must reach main) versus unreleased staging work that must NOT ship.
     Then rebase only the confirmed commits onto main:
     - if the hot-fix is exactly the commits unique to the branch past staging:
       `git rebase --onto origin/main origin/staging <branch>`;
     - otherwise an interactive `git rebase --onto origin/main <base> <branch>`, dropping the
       staging commits so only the confirmed fix remains.
       **Verify carefully before continuing:** `git log --oneline origin/main..<branch>` must
       show ONLY the confirmed hot-fix commits, and `git diff origin/main...<branch>` must
       match the intended fix and nothing else. Show this to the user and confirm. This branch
       was rebased — remember that for step 5.

## If the branch was rebased, re-run CI before merging

5. **Did step 4 rewrite the branch (a rebase happened)?**
   - **No (already on main):** continue to "Land on main".
   - **Yes:** the rewritten branch must pass CI on its PR before it can ship. Force-push it
     (`git push --force-with-lease origin <branch>`), ensure the hot-fix → main PR exists
     (create it if missing — same as step 7), then **STOP and wait for the PR's CI/CD to
     finish** (`gh pr checks <n>`). Tell the user to come back and re-run `/gitflow:hotfix <branch|n>`
     once the checks pass — on the re-run, step 4 finds the branch already on main and goes
     straight to landing. (Alternatively, if the user prefers, I can poll until the checks go
     green and then continue in the same run.)

## Land on main

6. Show the diff that will ship (`git diff origin/main...<branch>`) and confirm.
7. **A `hot-fix-*`→main PR is REQUIRED** (everyone reviews the fix). Check it exists
   (`gh pr list --head <branch> --base main`); if not, confirm and create it
   (`git push -u origin <branch>` → `gh pr create --base main`). Ask the user to get a quick
   review, then — on their confirm — merge with the **`hotfix:` subject prefix forced**
   (GitHub's default subject is just the PR title, so override it here):
   ```bash
   # ensure a meaningful body: gh pr view <n> --json body -q .body
   # if empty, draft from `git diff origin/main...<branch>`, confirm, gh pr edit <n> --body "..."
   gh pr merge <n> --squash \
     --subject "hotfix: <summary> (#<n>)" \
     --body "<PR description>"
   ```
   (If `gh` is unavailable, local fallback only with explicit OK since it bypasses review:
   `git checkout main && git pull --ff-only origin main && git merge --squash <branch>`,
   then commit with BOTH subject and body so it mirrors what GitHub's squash button
   produces — never a subject-only commit:
   - **subject** = `hotfix: <summary>`, appending ` (#<n>)` only if a PR number is known;
   - **body** = the PR description if one exists, otherwise the branch's commit messages as
     GitHub lists them (`git log origin/main..<branch> --reverse --format='* %s%n%n%b'`).
     e.g. `git commit -m "hotfix: <summary> (#n)" -m "<body>"`. Then push main.)
8. Tag the hotfix (annotated). Tags follow **semver + date**:
   `v<major>.<minor>.<patch>-<YYYYMMDD>`. A hotfix ALWAYS bumps the PATCH number from the
   last tag on main — no version question to ask. So the first hotfix after release
   `v1.1.0-20260610` is `v1.1.1-<today>`, a second hotfix on the same release is
   `v1.1.2-<today>`:
   ```bash
   # last semver tag reachable from HEAD (a prior hotfix tag counts — keep bumping from it):
   LAST_TAG=$(git tag --merged HEAD --list 'v*' --sort=-v:refname | head -1)
   BASE=${LAST_TAG%-*}                            # strip the date suffix: v1.1.0-20260610 -> v1.1.0
   IFS=. read -r MAJOR MINOR PATCH <<< "${BASE#v}"
   TAG="v${MAJOR}.${MINOR}.$((PATCH+1))-$(date +%Y%m%d)"
   git tag -a "$TAG" -m "Hotfix $TAG (on ${LAST_TAG})"
   ```
   If no `v*` tag exists yet, STOP and ask the user what version to start from — never
   invent one. Show `LAST_TAG` and the computed `TAG` to the user before tagging, then
   create and push the tag. (Phase 1: this tag is an experimental marker only — it does not
   trigger deployment.)
9. Push main + tag: show `git log --oneline origin/main..main`, then
   `git push origin main --follow-tags`. Confirm.

## Reconcile staging (REQUIRED — don't skip)

10. main now has the hotfix commit that staging lacks — the next release won't ff until
    staging is rebased onto main. Immediately run the `/gitflow:repair-staging` flow now:
    rebase staging onto main, dropping the hotfix's twin if it was already cherry-picked.
    **If the rebase conflicts** (the hotfix touched the same lines as unreleased staging
    work), confirm with the user that I'll try to resolve it; attempt it; if I struggle,
    ask the user to resolve in their editor, then continue. Tell the user this reconcile is
    part of the hotfix, not optional.
11. After staging is repaired & pushed, prepare the team announcement (old staging tip +
    "`git pull --rebase` on local staging; for an in-flight feature branch, run
    `/gitflow:sync-branch <old-staging-tip>` — or rebase by hand with
    `git rebase --onto origin/staging <old-staging-tip> <branch>`").

## Hard rules

- The user already has the `hot-fix-*` branch; this command resolves it (from a branch name,
  a PR number, or by asking) and never creates the fix.
- Hotfix is a SINGLE squash commit on main; never a merge commit, never multiple commits.
- Squash subject is forced to `hotfix: <summary> (#n)`; body = PR description (drafted and
  written to the PR if empty).
- A hotfix from a staging-based branch must be rebased onto main first, shipping ONLY the
  commits the user confirms are the fix — never unreleased staging work. Verify the rebased
  branch contains exactly those commits before merging.
- If the branch was rebased, force-push it and wait for the PR's CI/CD to pass before
  merging — re-run `/gitflow:hotfix` once green (or let me poll until green).
- Tag format is `v<major>.<minor>.<patch>-<YYYYMMDD>`; a hotfix always bumps PATCH from the
  last tag on main. Releases bump minor/major — a hotfix never does.
- Do not delete the hot-fix branch on merge (no `--delete-branch`).
- The staging repair (step 10) is mandatory — leaving it undone breaks the next release's ff.
- Confirm the shipping diff and every push; operate from repo root; fetch before deciding.
- Conflict rules: same as the shared rules — explain, propose, ask on non-trivial, never silently pick a side.
