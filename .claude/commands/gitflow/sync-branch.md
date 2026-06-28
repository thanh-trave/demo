---
description: Rebase my in-flight feature branch onto the new staging after a hotfix force-push (Phase 1, local-driven)
allowed-tools: Bash(git:*), Bash(cd:*), Bash(pwd:*)
---

# /gitflow:sync-branch

After a hotfix, `staging` is rebased onto `main` and force-pushed (its commits get new SHAs).
If you have a feature branch that was cut from the OLD staging, this rebases it cleanly onto
the NEW staging so your eventual PR has a clean diff and no foreign commits.

Optional argument — the old staging tip SHA from the team announcement (helps when the fork
point can't be auto-detected): $ARGUMENTS

## Preflight (ALWAYS)

1. `git rev-parse --show-toplevel`; if not a repo, stop. `cd` to root.
2. Capture the current branch (`git branch --show-current`) and `git status --porcelain`.
3. **Confirm this is a feature branch** (`feature-*`, `feat-*`, `fix-*`). REFUSE on
   `staging`/`main`/`dev`/detached HEAD — for a local `staging`, the fix is simply
   `git pull --rebase`, not this command. If unsure which branch, ask.
4. `git fetch origin staging` so we rebase onto the current remote staging.

## Short-circuit

5. If `origin/staging` is already an ancestor of HEAD
   (`git merge-base --is-ancestor origin/staging HEAD`), the branch is already current —
   report "nothing to do" and stop.

## Safety net

6. Create a backup ref before touching anything: `git branch -f backup/<branch> HEAD`, and
   tell the user how to restore (`git reset --hard backup/<branch>`). The reflog also keeps
   the old state ~90 days.

## Rebase

7. **Determine the fork boundary.** Use the SHA from $ARGUMENTS if given; otherwise try
   `git merge-base --fork-point origin/staging HEAD`. Show the user the commits that will be
   replayed (`git log --oneline <boundary>..HEAD`, or `origin/staging..HEAD` if no boundary)
   so they see exactly what's theirs.
8. **Rebase onto the new staging:**
   - With a boundary: `git rebase --autostash --onto origin/staging <boundary> <branch>`.
   - Without one: `git rebase --autostash origin/staging` (patch-id auto-drops the old
     staging twins that already exist on the new staging).
9. **On conflict:** confirm with the user that I'll try to resolve it. Per stop, distinguish:
   - A **foreign commit** (an old-staging commit whose twin is already on the new staging) →
     it should drop out; if it's conflicting, the boundary was likely wrong — abort and
     re-run with the announced old tip (`/gitflow:sync-branch <OLD_TIP>`).
   - **Your own commit** colliding with the hotfix → show both sides and attempt a
     resolution, confirming non-trivial choices. If I struggle, STOP and ask the user to
     resolve in their editor (`git add -A && git rebase --continue` when done). Never
     silently pick a side.
   - If a plain rebase (no boundary) conflicts on commits that clearly aren't yours,
     `git rebase --abort` and ask the user for the old staging tip, then redo with `--onto`.

## Finish

10. Show the result (`git log --oneline -8`) so the user can sanity-check that only their
    work sits on top of current staging. Then **ask before pushing**:
    `git push --force-with-lease origin <branch>` (force is needed because the branch was
    rebased; `--with-lease` protects a shared branch — never bare `--force`). If the lease is
    rejected, someone else pushed to the branch — re-fetch and reconcile, don't override.
11. Tell the user they can delete the backup ref (`git branch -D backup/<branch>`) once happy.

## Hard rules

- Feature/fix branches only — never run on `staging`/`main`.
- Always create the backup ref and show the replay list before rebasing.
- Confirm before trying conflict resolution; hand off to the user's editor if I struggle.
- Always `--force-with-lease`, never bare `--force`; a rejected lease means STOP and re-fetch.
- A branch that isn't synced still merges fine later (squash lands clean code) — this is
  hygiene (clean diffs, early conflict surfacing), so it's safe to run right before opening
  the PR rather than immediately.
