---
description: Release — fast-forward main to staging (or a partial-release branch) and tag (Phase 1, local-driven)
allowed-tools: Bash(git:*), Bash(gh:*), Bash(cd:*), Bash(pwd:*)
---

# /gitflow:release

Promote to production by **fast-forwarding `main`** to the release source, so main and the
release source stay one identical linear chain. The source is normally `staging`; for a
**partial release** the user manually prepares a `staging`-like branch (e.g. `staging-2`)
holding only the subset to ship and passes it as the argument. Phase 1: performed locally;
everyone can push to main, so the safety comes from careful confirmation here.

Optional argument — the release source branch. Defaults to `staging`. Pass a manually
prepared partial-release branch like `staging-2` to ship only that subset: $ARGUMENTS

## Preflight (ALWAYS)

1. `git rev-parse --show-toplevel`; if not a repo, stop. `cd` to root.
2. **Remind the user to tell the team to STOP pushing to `staging` and `main`** for the
   duration of the release — a branch moving mid-release is the main avoidable hazard. Ask
   them to confirm the team has been notified before proceeding.
3. `git fetch origin --tags` so `origin/main`, `origin/staging` and any partial-release
   branch are current. Capture current branch and `git status --porcelain`. **Resolve the
   source:** `<source>` = the branch in $ARGUMENTS if given (e.g. `staging-2`), otherwise
   `staging`. If a branch was passed, verify it exists on origin; if not, STOP and ask.
4. If the worktree is dirty, stash with confirmation and remember to restore at the end —
   a release must not sweep in uncommitted local edits.

## Require a review PR (everyone reviews before release)

5. **A `<source>`→main PR is REQUIRED.** Check it exists
   (`gh pr list --head <source> --base main`).
   - **No PR:** confirm with the user, then `gh pr create --base main --head <source>`
     (title like "Release - <date>", or "Partial release - <date>" for a partial-release
     branch). Ask them to get it reviewed before continuing, or to confirm they want to
     proceed now.
   - **PR exists:** before proceeding, **check the PR's CI/CD has finished**
     (`gh pr checks <n>`). If any check is still running/pending, STOP and tell the user to
     wait for the checks to finish, then re-run `/gitflow:release` — never release on top of an
     in-flight or failing pipeline. Proceed only once all checks have completed and the user
     confirms the PR's been reviewed.

## Verify the fast-forward is possible (the core safety check)

6. Confirm `origin/main` is an ancestor of `origin/<source>`:
   `git merge-base --is-ancestor origin/main origin/<source>`.
   - **If yes:** good — the release is a clean ff.
   - **If NO:** main has diverged (an unrepaired hotfix, or a stray commit on main). Do NOT
     force anything. Show what diverged (`git log --oneline origin/main ^origin/<source>`),
     then **confirm with the user that I'll rebase `<source>` onto main first** (the
     `/gitflow:repair-staging` operation, applied to `<source>`): rebase, trying to resolve conflicts
     where possible — foreign/twin commits skipped, genuine collisions resolved with
     confirmation, and if I struggle, ask the user to resolve in their editor. Force-push
     `<source>` (`--force-with-lease`). **Then STOP — do not continue the release in this
     run.** The force-push rewrote `<source>`, so its CI/CD must run green on the new tip
     before anything ships. Tell the user to wait for the checks to finish successfully and
     then **run `/gitflow:release` again** — the re-run will see `origin/main` is now an ancestor and
     do a clean ff. Also prepare the team announcement now (old `<source>` tip + "`git pull
--rebase` on local `<source>`"), since `<source>` was just rewritten.
7. Show the user exactly what will ship: `git log --oneline origin/main..origin/<source>`.
   Ask for explicit confirmation to release these commits.

## Execute

8. Fast-forward main locally and verify identity:
   `git checkout main && git pull --ff-only origin main`
   `git merge --ff-only origin/<source>`
   (this must succeed without creating a commit; if git tries to make a merge commit,
   something is wrong — stop and re-check step 6).
9. Create the release tag (annotated). Tags follow **semver + release date**:
   `v<major>.<minor>.<patch>-<YYYYMMDD>` (e.g. `v1.1.0-20260612`). A normal release bumps
   the MINOR number and resets patch to 0, but the version is the user's call — **ALWAYS
   show the proposed version and get the user's confirmation (or their override, e.g. a
   major bump) before tagging.**
   ```bash
   # last semver tag reachable from HEAD (hotfix tags count — their patch bumps are part of the line):
   LAST_TAG=$(git tag --merged HEAD --list 'v*' --sort=-v:refname | head -1)
   # propose: MINOR+1, patch reset to 0 (e.g. v1.0.1-20260601 -> v1.1.0); if no v* tag yet, propose v1.0.0
   # >>> show LAST_TAG + the proposed version, ask the user to confirm or override <<<
   VERSION=v1.1.0                      # the version the user confirmed
   TAG="${VERSION}-$(date +%Y%m%d)"
   git tag -a "$TAG" -m "Release $TAG"
   ```
   Tell the user the tag name.
10. Push: show `git log --oneline origin/main..main` (should equal step 7's list), then
    `git push origin main --follow-tags`. Plain push — a ff never needs force. Confirm success.

## Reconcile staging after a partial release

11. **If `<source>` was `staging`:** main and staging are now identical — nothing to
    reconcile. Restore any stash from step 4 and report the new `main` tip + tag.
12. **If `<source>` was a partial-release branch (e.g. `staging-2`):** main has moved ahead
    of `staging`, which still holds the work that wasn't shipped — so `origin/main` is no
    longer an ancestor of `origin/staging`. **`staging` must be rebased onto main again:**
    run the `/gitflow:repair-staging` flow now (rebase `staging` onto main, force-push with lease,
    handle conflicts with confirmation). It produces its own before/after and announcement.
    Then restore any stash and report the new `main` tip + tag.

## Hard rules

- Before starting, the team must be told to stop pushing to `staging`/`main`.
- The source is `staging` by default, or a manually prepared partial-release branch (e.g.
  `staging-2`) passed as the argument. After a partial release, `staging` MUST be repaired
  (rebased onto main) — don't leave it diverged.
- A `<source>`→main review PR must exist before releasing, and its CI/CD must have finished —
  never release while the PR's checks are still running or failing.
- A release is ALWAYS a fast-forward. If it can't ff, rebase the source onto main first
  (with confirmation), never `--force` main and never a merge commit on main.
- After rebasing + force-pushing the source, STOP and wait for its CI/CD to pass; the
  release continues only on a re-run of `/gitflow:release` once `origin/main` is an ancestor again.
- Never tag or push until the user has seen and confirmed the shipping commit list.
- Tag format is `v<major>.<minor>.<patch>-<YYYYMMDD>`. Default proposal is a minor bump, but
  the version number is ALWAYS confirmed with the user — never tag a release unprompted.
- The review PR is kept in DRAFT by the `guard-cross-merge` workflow on purpose — never try
  to mark it ready or merge it from GitHub; the release lands as this ff push, after which
  GitHub closes the PR as merged automatically.
- Operate from repo root; fetch before deciding; report the new main tip + tag at the end.
- Phase 1: the tag is an experimental marker only — it does NOT trigger deployment and does
  not change the existing CI trigger. Just create and push it; deployment happens however it
  does today.
