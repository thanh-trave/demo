# Workflow files — one-time setup checklist (Approach A)

Copy the five `.yml` files into `.github/workflows/`. Then wire up the platform pieces they depend on — **the order matters** (rulesets last, or you'll lock yourself out before the bot exists):

## 1. Bot account + PAT
- Create a machine user (e.g. `trave-release-bot`), add it to the repo with **Write** access.
- Generate a **fine-grained PAT**: this repo only, permissions `Contents: Read & write`, `Pull requests: Read`.
- Save it as an **environment secret** named `RELEASE_BOT_TOKEN` inside the `production-release` environment (Settings → Environments → production-release → secrets), **not** as a repo secret. Why: the bot's ruleset bypass is all-or-nothing — it *can* force-push main — and environment-scoping means the token is only injected into jobs gated by reviewer approval, so no unreviewed workflow can ever read it.
- Replace the placeholder bot name/email in the yml files (`trave-release-bot`, `release-bot@trave.example`) — the `tag-hotfix.yml` actor check must match the real account name exactly.
- ⚠ `tag-hotfix.yml` does NOT declare the environment (hotfix tags shouldn't wait for approval at 2am). Give it its own secret: a second, even narrower PAT (or the same one) stored as a repo secret used only there — tag pushes can't rewrite branches, so its blast radius is small.

## 2. Environments (Settings → Environments)
- **`production-release`** — add Required reviewers (release managers). Gates the `release.yml` and `rebase-staging.yml` buttons.
- **`production`** — required reviewers for the prod deploy (or reuse `production-release` in `deploy.yml` if one human gate is enough).
- **`qa`**, **`uat`** — no reviewers needed; they exist for deployment history/scoped secrets.

## 3. Azure (for `deploy.yml`)
- OIDC federated credential for the repo → secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.
- Replace placeholders: registry (`traveacr.azurecr.io`), image (`trave-api`), resource group (`trave-rg`), app names (`trave-api`, `-qa`, `-uat`).

## 4. Rulesets — do this LAST (import the JSON files)
Import `ruleset-main.json` and `ruleset-staging.json`: Settings → Rules → Rulesets → **New ruleset ▾ → Import a ruleset**. CLI alternative:
```bash
gh api repos/OWNER/REPO/rulesets --method POST --input ruleset-main.json
gh api repos/OWNER/REPO/rulesets --method POST --input ruleset-staging.json
```

What they encode — **main**: require PR (1 approval, **squash only**), required check **`guard`**, linear history required, force pushes & deletion blocked. **staging**: require PR (squash only), linear history, force pushes & deletion blocked. Deliberately absent from main: a blanket "restrict updates" rule — it would also block humans from squash-merging `hot-fix-*` PRs; the guard check is what blocks release PRs instead.

**⚠ Bypass actors must be added manually after import.** `bypass_actors` is empty in the JSON because rulesets can't list individual user accounts as bypass actors, and team/app IDs are instance-specific numbers. After importing, open each ruleset → **Bypass list** → add ONE of:
- a **team** containing only `trave-release-bot` (needs a GitHub org — recommended), or
- a **deploy key** (generate an SSH key, add as deploy key with write access, switch the workflows from PAT to SSH push), or
- a **GitHub App** if you later replace the PAT with an app installation.

Without the bypass actor, `release.yml` and `rebase-staging.yml` will fail on push — that's the expected symptom of skipping this step. Also: add your CI's job context next to `guard` in main's required checks once you know its name (`integration_id` 15368 = GitHub Actions).

**Optional — merge queue on staging (busy-Thursday pile-ups):** edit the imported `staging-protection` ruleset → add the **"Require merge queue"** rule, merge method **squash**. It serializes simultaneous PRs automatically: each is CI-tested against staging *plus* the PRs queued ahead of it, so a stale green check can never break staging. Prerequisite: your CI workflow must also trigger on the queue's synthetic event, or queued PRs hang forever:
```yaml
on:
  pull_request:
  merge_group:    # ← required for merge queue; harmless to add now even if the queue stays off
```
Reasonable policy: add the trigger line today, flip the ruleset rule on the first Thursday the pile-up actually hurts.

## 5. Repo settings (Settings → General → Pull Requests)
- Default squash commit message: **Pull request title** (prevents stale-message leaks after rebases — Case A4).
- Enable "Allow squash merging"; disable "Allow merge commits" and "Allow rebase merging".

## What each file does
| File | Trigger | Purpose |
|---|---|---|
| `release.yml` | Manual button + approval | ff-only staging/release-* → main, create `vYYYY-MM-DD` tag; if source was a `release-*` branch, a chained job repairs staging in the same approved run |
| `rebase-staging.yml` | Manual button + auto on hotfix push to main (both gated by approval) | The repair ritual (post-hotfix A3b); tries ff first, falls back to rebase + force-with-lease; auto-trigger queues at the gate and notifies reviewers |
| `release-guard.yml` | Every PR to main | Required check: red for staging/release-*/unknown heads (blocks UI merge), green for hot-fix-* |
| `tag-hotfix.yml` | Push to main (non-bot) | Auto-derive `vDATE-hfN` tag → triggers prod deploy |
| `deploy.yml` | Push staging / release-* / tag v* | Build image (tagged ref+SHA) → QA / UAT / Prod on Azure Container Apps |

## First run order (migration)
1. Repo settings + secrets + environments (steps 1–3, 5).
2. **Case A8 one-time cleanup** of the current X1/X2 state (terminal, before rulesets block you).
3. Add the workflow files; verify `release-guard` turns green/red correctly on a test PR.
4. Apply rulesets (step 4).
5. Next Thursday: run the Release button. 🎉