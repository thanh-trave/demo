---
description: Repair staging by rebasing it onto main after a hotfix (Phase 1, local-driven)
allowed-tools: Bash(git:*), Bash(cd:*), Bash(pwd:*)
---

# /gitflow:repair-staging

Restore the invariant "main is an ancestor of staging" by rebasing staging's unreleased
commits on top of main, then force-pushing staging. Run this after a hotfix lands on main,
or after a partial / release-branch release. Phase 1: everyone can force-push, so this is
done locally — carefully.

Optional argument — the known-good old staging tip SHA, if you have one: $ARGUMENTS

## Preflight (ALWAYS)

1. `git rev-parse --show-toplevel`; if not a repo, stop. `cd` to root.
2. `git fetch origin main staging`. Capture current branch + `git status --porcelain`.
3. If the worktree is dirty, stash with confirmation (restore at the end). This operates on
   the shared `staging` branch, not the user's feature work.

## Check whether a repair is even needed

4. If `git merge-base --is-ancestor origin/main origin/staging` is already true, staging is
   fine — report "no repair needed" and stop.

## Prepare a local staging that matches origin

5. `git checkout -B staging origin/staging` (local staging now == origin/staging).
6. **Record and announce the old tip:** `OLD_TIP=$(git rev-parse HEAD)`. Show it to the
   user — it's the pre-rebase staging tip, useful as a recovery reference and for anyone
   rebasing an in-flight feature branch off the old staging. Create a safety ref:
   `git branch -f backup/staging-<date> HEAD`.

## Try the cheap path, then rebase

7. `git merge --ff-only origin/main` — if staging hadn't moved, this just fast-forwards;
   no force-push needed. If it succeeds, skip to step 10.
8. Otherwise rebase: `git rebase origin/main`. **On the first conflict, confirm with the
   user that I'll try to resolve it.** Then, per stopped commit:
   - **Foreign/twin** (a hotfix already cherry-picked into staging): confirm by checking
     main for the equivalent change, then `git rebase --skip`. Explain which main commit
     supersedes it.
   - **Genuine unreleased work** colliding with the hotfix: show both sides and attempt a
     resolution that preserves BOTH the hotfix and the feature intent, confirming before
     applying anything non-trivial. **If I struggle or they're fundamentally incompatible,
     STOP and ask the user to resolve in their editor** (`git add -A && git rebase
--continue` when done) — never silently pick a side; a genuine clash is a code
     decision for the team.
9. Verify the result: `git log --oneline origin/main..HEAD` should show only unreleased
   staging work — no merge commits, no duplicated hotfix.

## Confirm and push

10. Show the before/after clearly: old tip vs new tip, and `git range-diff origin/main
origin/staging HEAD` so the user sees that no work was lost. Get explicit confirmation.
11. Force-push with lease (protects against someone merging to staging mid-repair):
    `git push --force-with-lease origin staging`. If the lease is rejected, someone pushed
    to staging — re-fetch and redo from step 5; never use bare `--force` to override.
12. Restore any stash. Prepare the announcement: the OLD_TIP + the team one-liner —
    "`git pull --rebase` on local staging; for an in-flight feature branch, run
    `/gitflow:sync-branch <OLD_TIP>` — or rebase by hand with
    `git rebase --onto origin/staging <OLD_TIP> <branch>`". Tell the user they can delete
    the backup ref once everyone has resynced.

## Hard rules

- Always `--force-with-lease`, never bare `--force`. A rejected lease means STOP and re-fetch.
- Always create the backup ref and capture OLD_TIP before rebasing.
- Never resolve a conflict by discarding a side silently; foreign twins are skipped, real
  conflicts are resolved with confirmation, design clashes are escalated.
- Operate from repo root; fetch first; confirm before the force-push.
