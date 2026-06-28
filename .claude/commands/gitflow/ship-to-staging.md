---
description: Squash-merge the current feature branch into staging (Phase 1, local-driven)
allowed-tools: Bash(git:*), Bash(gh:*), Bash(cd:*), Bash(pwd:*)
---

# /gitflow:ship-to-staging

Squash-merge my feature work into `staging`, keeping staging a clean linear chain of
one-commit-per-feature. Phase 1: everyone can push to staging, so this is done locally
with careful confirmation rather than through protected workflows.

Optional argument — the feature branch to ship (defaults to the current branch): $ARGUMENTS

## Preflight (ALWAYS do this first, every command)

1. **Locate the repo.** Run `git rev-parse --show-toplevel`. If it fails, I'm not in a git
   repo — stop and tell the user. `cd` to the toplevel so all commands run from root
   regardless of whether the user invoked from a subfolder.
2. **Identify state.** Capture current branch (`git branch --show-current`), and run
   `git status --porcelain` and `git stash list`.
3. **Resolve which branch to ship.** The branch may be passed as $ARGUMENTS — the user can
   be standing on one branch while shipping another.
   - **Branch given as $ARGUMENTS:** ship that branch (no need to be on it). Verify it
     exists and is a work branch (`feature-*`, `feat-*`, `fix-*`); if not, STOP and ask.
   - **No argument:** default to the current branch, but **confirm with the user that this
     is the branch they mean before going further** — don't assume. If the current branch
     is `staging`/`main`/`dev` or a detached HEAD, STOP and ask which feature branch to
     ship; never guess.
4. **Sync the picture.** `git fetch origin` so local refs reflect origin before any
   decision — in particular the latest `origin/staging`.

## Procedure

5. **Handle uncommitted work.** If the worktree is dirty: show the changes
   (`git status` + `git diff --stat`), ask whether to commit them into this feature
   (offer to write a commit message) or stash them. Never discard without asking.
6. **Rebase onto current staging.** Check out the resolved branch if you're not already on
   it (`git switch <branch>`). Compare it against `origin/staging`. If
   `origin/staging` has advanced, rebase the branch onto it (`git rebase
origin/staging`). **If the rebase conflicts:** confirm with the user that I'll try to
   resolve it; attempt the resolution; if I struggle (non-trivial logic, or I'm not
   confident), STOP and ask the user to resolve in their editor, then continue once they
   say it's done (`git rebase --continue`). This guarantees a clean squash with no surprise
   conflicts at merge time.
   **If the rebase rewrote the branch and it already exists on `origin`** (a PR is open, or a
   remote tracking branch exists), the remote tip now lags the rebased local branch — so
   **force-push it with lease before merging so the PR's diff and CI reflect exactly what will
   be squashed:** confirm, then `git push --force-with-lease origin <branch>` (its checks
   re-run on the new tip). If the branch isn't on `origin` yet, don't push here — step 7
   pushes it when it opens the PR. Never bare `--force`; a rejected lease means someone else
   pushed — STOP and re-fetch.
7. **A PR to staging is REQUIRED** (everyone reviews before merge). Check whether a PR from
   this branch to `staging` already exists (`gh pr list --head <branch> --base staging`).
   - **No PR:** **generate the PR title** — don't ask the user to supply one. Derive a
     concise, imperative title from the branch's commits and diff
     (`git log origin/staging..<branch>` + `git diff --stat origin/staging..<branch>`). Look
     for a Linear id (pattern `[A-Z]{2,}-[0-9]+`, e.g. `TRA-123`) first in the branch name,
     then in the commit messages; if one is found, format the title as `<linear-id>: <title>`,
     otherwise just the title. **Show the proposed title and get the user's OK (or their
     edit) before creating the PR.** Draft a body from the commits/diff too, but **don't ask
     the user to confirm the body**. Then `git push -u origin <branch>` and
     `gh pr create --base staging --title "<title>" --body "<drafted body>"`. Then either
     tell the user to request review from a teammate, OR — only if the user explicitly says
     so — proceed to merge the PR.
   - **PR exists:** first make sure the rebased branch has been pushed (step 6) so the PR's
     diff and CI reflect exactly what will be squashed — if the local branch is ahead of its
     remote after a rebase, force-push it with lease before continuing. Then proceed to the
     merge step once the user confirms it's reviewed/approved.
8. **Squash-merge the PR** (after the user confirms it's reviewed). With the repo default
   "Pull request title and description", the squash commit is:
   - **subject** = `<PR title> (#<n>)` (GitHub default — don't override),
   - **body** = the PR description.
     So before merging, **ensure the PR description is meaningful**:
     `gh pr view <n> --json body -q .body`. If it's empty or trivial, draft a concise summary
     from the branch (`git log origin/staging..<branch>` + `git diff --stat`) and write it
     into the PR (`gh pr edit <n> --body "<summary>"`) — no need to confirm the body with the
     user. Then merge **without deleting the branch**: `gh pr merge <n> --squash`. Confirm
     staging's new tip afterward.
     (If `gh` is unavailable, local fallback only with explicit OK since it bypasses review:
     `git checkout staging && git pull --ff-only origin staging && git merge --squash <branch>`,
     then commit so the result mirrors GitHub's squash button:
   - **subject** = if the branch has a single commit, that commit's subject; if it has
     several, the PR title you generated. Append ` (#<n>)` only if a PR number is known,
     otherwise omit it.
   - **body** = the branch's commit messages exactly as GitHub lists them
     (`git log origin/staging..<branch> --reverse --format='* %s%n%n%b'`).
     Don't push beyond what the user confirms; don't delete the branch.)
9. **Keep the branches.** Do not delete the feature branch — neither the local branch nor
   the remote (`origin`) one. Just report staging's new tip and that the merge is done.

## Conflict resolution rules (apply in every command)

- Show me each conflicted file and the competing hunks; explain what each side intends.
- Propose a resolution for trivial/mechanical conflicts and apply after a quick confirm.
- For non-trivial logic conflicts, ASK before resolving — never silently pick a side,
  never `checkout --ours/--theirs` on real code without explaining and confirming.
- If the conflict implies a real design decision, STOP and surface it as a human decision.

## Hard rules

- Always operate from the repo root; always `git fetch` before deciding anything.
- Never force-push a feature branch that others may share without `--force-with-lease` and a heads-up.
- After rebasing a branch that already has an open PR, force-push it (`--force-with-lease`)
  so the PR diff and CI match what gets squashed — never squash-merge a PR whose remote tip
  predates the rebase.
- Squash only into staging — one commit per feature. Never create a merge commit here.
- The PR title is generated (not asked for) and confirmed with the user; format it as
  `<linear-id>: <title>` when a Linear id is found, else just the title.
- On the `gh` path the squash commit is `<PR title> (#n)` + the PR-description body; if the
  description is empty, draft one and write it to the PR before merging (no body confirmation
  needed). The `gh`-unavailable fallback instead mirrors GitHub's squash UI: commit-or-PR
  title subject + commit-message body.
- Never delete the feature branch (local or remote) after merge.
- Confirm with me before any push.
- If anything is ambiguous (which branch, which commits, dirty state), ask — don't assume.
