# Shared GitHub Actions Workflows

Centralized, reusable GitHub Actions workflows for deploying and content‑syncing projects across the organization. Each project repo calls these workflows with a small caller file and a handful of secrets — no copy‑pasted YAML, no per‑repo drift.

---

## What's in here

| Workflow | File | Purpose |
| --- | --- | --- |
| **Deploy** | `.github/workflows/deploy.yml` | SSH into the target server, `git pull` the branch, run `composer install` / `npm run build`, notify Slack. Triggered by the caller on `push` to `production` / `staging` / `dev`. |
| **Content Push** | `.github/workflows/content-push.yml` | SSH into the server, commit any server‑side content changes, push them back to Git, then rebuild. Triggered by the caller on a schedule. Prevents server content from being overwritten by a later deploy. |

Both workflows are defined with `on: workflow_call`, which means they can only be invoked by another workflow — they don't run on their own.

---

## Why a central repo

- **One source of truth.** Fix a bug or improve the deploy script once; every project picks it up on the next run.
- **Per‑repo flexibility stays per‑repo.** Triggers (`push`, `schedule`, `workflow_dispatch`), branch lists, cron expressions, and dispatch inputs live in the caller, so each project can tune them without touching shared code.
- **Versioned rollout.** Callers pin to a tag (`@v1`); breaking changes ship as `@v2` and projects migrate when ready.
- **Least‑privilege auth.** A single GitHub App grants short‑lived, repo‑scoped tokens instead of long‑lived personal access tokens.

---

## Quick start: adding these workflows to a project

### 1. Make sure the project has the required secrets

Add these as **repository secrets** in every project repo (Settings → Secrets and variables → Actions). Organization secrets are not used here because the Free org plan does not allow private repos to read them — see [Secrets distribution](#secrets-distribution) below for a one‑command script that pushes them to every repo at once.

| Secret | Required | Description |
| --- | --- | --- |
| `SSH_USER` | always | SSH username on the target servers |
| `SSH_PASS` | always | SSH password (consider migrating to an SSH key) |
| `PROD_SERVER` | deploy + content push | Hostname/IP of the production server |
| `STAGE_SERVER` | deploy | Hostname/IP of the staging server |
| `DEV_SERVER` | deploy | Hostname/IP of the dev server |
| `DEPLOY_APP_ID` | always | Numeric ID of the deploy GitHub App |
| `DEPLOY_APP_PRIVATE_KEY` | always | Full `.pem` contents of the App's private key |
| `SLACK_WEBHOOK_URL` | always | Incoming webhook URL for Slack notifications |

### 2. Add a caller workflow for deploys

Create `.github/workflows/deploy.yml` in the project repo:

```yaml
name: Deploy

on:
  push:
    branches: [production, staging, dev]
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to deploy"
        required: true
        default: "production"
      reason:
        description: "Reason for deployment"
        required: false

jobs:
  call:
    uses: your-org/workflow-templates/.github/workflows/deploy.yml@v1
    with:
      repo_slug: your-org/your-repo
      # Optional overrides — only set if non-default:
      # prod_path: /html/production
      # stage_path: /html/staging
      # dev_path: /html/dev
      # run_composer: true
      # run_npm_build: true
    secrets:
      SSH_USER:               ${{ secrets.SSH_USER }}
      SSH_PASS:               ${{ secrets.SSH_PASS }}
      PROD_SERVER:            ${{ secrets.PROD_SERVER }}
      STAGE_SERVER:           ${{ secrets.STAGE_SERVER }}
      DEV_SERVER:             ${{ secrets.DEV_SERVER }}
      DEPLOY_APP_ID:          ${{ secrets.DEPLOY_APP_ID }}
      DEPLOY_APP_PRIVATE_KEY: ${{ secrets.DEPLOY_APP_PRIVATE_KEY }}
      SLACK_WEBHOOK_URL:      ${{ secrets.SLACK_WEBHOOK_URL }}
```

> **Use the explicit `secrets:` block, not `secrets: inherit`.**
> `inherit` only works when the caller and the reusable workflow live under the **same owner** (same user or same org). If your project repos are in one org and the workflow templates repo is under a different account, `inherit` silently forwards nothing and every required secret fails validation. Explicit forwarding works across owner boundaries and doubles as documentation of what the project actually depends on.

### 3. Add a caller workflow for content push

Create `.github/workflows/content-push.yml`:

```yaml
name: Content Push

on:
  schedule:
    - cron: '*/30 * * * *'   # every 30 minutes
  workflow_dispatch:

jobs:
  call:
    uses: your-org/workflow-templates/.github/workflows/content-push.yml@v1
    with:
      repo_slug: your-org/your-repo
      branch: production
      # remote_path: /html/production
    secrets:
      SSH_USER:               ${{ secrets.SSH_USER }}
      SSH_PASS:               ${{ secrets.SSH_PASS }}
      PROD_SERVER:            ${{ secrets.PROD_SERVER }}
      DEPLOY_APP_ID:          ${{ secrets.DEPLOY_APP_ID }}
      DEPLOY_APP_PRIVATE_KEY: ${{ secrets.DEPLOY_APP_PRIVATE_KEY }}
      SLACK_WEBHOOK_URL:      ${{ secrets.SLACK_WEBHOOK_URL }}
```

That's all you need in the project repo — two files, no shared logic to maintain.

---

## Inputs

### `deploy.yml`

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `repo_slug` | string | *required* | `owner/repo` used in the remote URL on the server |
| `prod_path` | string | `/html/production` | Absolute path on the production server |
| `stage_path` | string | `/html/staging` | Absolute path on the staging server |
| `dev_path` | string | `/html/dev` | Absolute path on the dev server |
| `runs_on` | string | `self-hosted` | Runner label |
| `run_composer` | boolean | `true` | Run `composer install` after pulling |
| `run_npm_build` | boolean | `true` | Run `npm install && npm run build` after pulling |

### `content-push.yml`

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `repo_slug` | string | *required* | `owner/repo` for the git remote on the server |
| `branch` | string | `production` | Branch to sync (server → git) |
| `remote_path` | string | `/html/production` | Absolute path on the production server |
| `runs_on` | string | `self-hosted` | Runner label |
| `run_composer` | boolean | `true` | Run `composer install` after sync |
| `run_npm_build` | boolean | `true` | Run `npm install && npm run build` after sync |

---

## Authentication: GitHub App, not PAT

These workflows authenticate to GitHub using a **GitHub App installation token** instead of a personal access token. The app is owned by the org, installed on the relevant repos, and mints a fresh token at the start of each job that expires in ~1 hour.

### Why

- **No human dependency.** A PAT is tied to a user; if they leave, every workflow breaks until the token is rotated. An app is an org‑owned bot identity.
- **Automatic short-lived tokens.** Each job gets a fresh token, valid ~1 hour, scoped only to the calling repo. Nothing to rotate manually.
- **Least privilege.** The app only has the permissions you explicitly granted, and only on the repos it's installed in.
- **Higher rate limits.** 15,000 req/hour per installation vs. 5,000 for a PAT.

### One‑time setup

1. Org **Settings → Developer settings → GitHub Apps → New GitHub App**.
2. Fill in:
   - **Name:** e.g. `acme-deploy-bot`
   - **Homepage URL:** any valid URL
   - **Webhook:** uncheck **Active**
3. **Repository permissions:**
   - **Contents:** Read and write
   - **Metadata:** Read-only (auto-selected)
   - **Workflows:** Read and write *only* if pushes will touch files under `.github/workflows/`
4. **Where can this GitHub App be installed?** → Only on this account.
5. Create the app, then on its settings page:
   - Copy the **App ID**.
   - Click **Generate a private key** and download the `.pem` file.
6. Left sidebar → **Install App** → install on the org → choose the project repos that should be deployable.
7. Add `DEPLOY_APP_ID` and `DEPLOY_APP_PRIVATE_KEY` to each project repo (see [Secrets distribution](#secrets-distribution) for a bulk script):
   - `DEPLOY_APP_ID` = the numeric App ID
   - `DEPLOY_APP_PRIVATE_KEY` = the full contents of the `.pem` file (including the `-----BEGIN/END-----` lines)

### How the workflows use it

At the top of each job, the workflow exchanges the App ID + private key for a short‑lived installation token via `actions/create-github-app-token`. The token is then used as the password in the git remote URL on the server:

```bash
# Username is the literal string "x-access-token"
git remote set-url origin \
  "https://x-access-token:${APP_TOKEN}@github.com/${REPO_SLUG}.git"
```

`x-access-token` is a fixed sentinel username GitHub expects for installation tokens — don't replace it with a real username.

### Commit attribution

Commits made by the workflow (e.g. the content‑push sync commit) are attributed to `acme-deploy-bot[bot]`. To make that explicit on the commit object itself:

```bash
git -c user.name="acme-deploy-bot[bot]" \
    -c user.email="<app-id>+acme-deploy-bot[bot]@users.noreply.github.com" \
    commit -m "chore(content): sync from server [skip ci]"
```

The exact noreply email for the bot can be found at `https://api.github.com/users/acme-deploy-bot[bot]`.

---

## Secrets distribution

This org is on the **Free plan**, where organization secrets are not visible to private repositories. All secrets are therefore set per repo. To keep that bearable, use the bulk script in this repo.

### Required per repo

- `SSH_USER`, `SSH_PASS`
- `PROD_SERVER` (+ `STAGE_SERVER` / `DEV_SERVER` for the deploy workflow)
- `DEPLOY_APP_ID`, `DEPLOY_APP_PRIVATE_KEY`
- `SLACK_WEBHOOK_URL`

### Bulk‑set the deploy app secrets

`scripts/sync-deploy-secrets.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Usage:
#   export DEPLOY_APP_ID=123456
#   export DEPLOY_APP_PRIVATE_KEY_FILE=~/secrets/acme-deploy-bot.private-key.pem
#   ./scripts/sync-deploy-secrets.sh your-org/repo-a your-org/repo-b ...

: "${DEPLOY_APP_ID:?set DEPLOY_APP_ID}"
: "${DEPLOY_APP_PRIVATE_KEY_FILE:?set DEPLOY_APP_PRIVATE_KEY_FILE to the .pem path}"

if [ "$#" -eq 0 ]; then
  echo "Pass one or more owner/repo arguments." >&2
  exit 1
fi

for repo in "$@"; do
  echo ">> $repo"
  gh secret set DEPLOY_APP_ID          --repo "$repo" --body "$DEPLOY_APP_ID"
  gh secret set DEPLOY_APP_PRIVATE_KEY --repo "$repo" < "$DEPLOY_APP_PRIVATE_KEY_FILE"
done
```

Run it (after `gh auth login`):

```bash
export DEPLOY_APP_ID=345678
export DEPLOY_APP_PRIVATE_KEY_FILE=~/keys/acme-deploy-bot.private-key.pem

./scripts/sync-deploy-secrets.sh \
  your-org/repo-a \
  your-org/repo-b \
  your-org/repo-c
```

Or keep the repo list in a file and pipe it in:

```bash
xargs -a scripts/deploy-repos.txt ./scripts/sync-deploy-secrets.sh
```

### Rotating the private key

1. Generate a new private key on the GitHub App's settings page.
2. Re-run `sync-deploy-secrets.sh` with the new `.pem`.
3. Delete the old key on the App settings page.

No code changes, no per‑repo clicking.

### Why not org secrets

GitHub's Free org plan limits organization secrets to public repos. The two paid alternatives are:

- **Upgrade to Team** (~$4/user/month) — unlocks org secrets for private repos.
- **Use a GitHub Environment** per repo — same per‑repo cost as repo secrets, but adds approval gates if you want them.

For a small fleet, the bulk script above is the simplest path and keeps the App rotation story to a single command.

---

## Versioning

Callers pin to a tag, not a branch:

```yaml
uses: your-org/workflow-templates/.github/workflows/deploy.yml@v1
```

Release rules used here:

- **`v1`, `v2`, ...** — major tags. Always point at the latest non‑breaking commit on that line.
- **`v1.2.3`** — immutable release tags. Pin to one of these for maximum stability.
- **`main`** — do **not** use in callers. Reserved for development.

When making a breaking change (renaming an input, removing a default, changing required secrets), bump the major and update callers deliberately.

---

## Access from other repos

This is a private repo, so callers can only use these workflows if access is explicitly granted:

**Settings → Actions → General → Access** on this repo → select  
**"Accessible from repositories in the 'your-org' organization"**.

Without this, callers in other repos will fail with a 404 when resolving `uses:`.

---

## Local changes and contributions

- Edit on a feature branch, open a PR against `main`.
- Test the change end‑to‑end by pointing a sandbox project's caller at the feature branch:

  ```yaml
  uses: your-org/workflow-templates/.github/workflows/deploy.yml@my-feature
  ```

- After merge, move the appropriate major tag (`git tag -f v1 && git push -f origin v1`) so all callers pick up the change on their next run.
- For breaking changes, cut a new major instead of force‑moving the existing one.

---

## Troubleshooting

**`Error: Resource not accessible by integration`**  
The app doesn't have the permission the workflow is asking for, or isn't installed on the repo. Re‑check repository permissions on the app, then re‑install it so the new permissions take effect.

**`remote: Permission to owner/repo.git denied to acme-deploy-bot[bot]`**  
The app is installed, but not on this specific repo. Org → Settings → GitHub Apps → Configure → add the repo.

**`fatal: unable to access ... The requested URL returned error: 403`** during a workflow file push  
Grant **Workflows: write** to the app.

**Caller fails with `workflow was not found` or "reference … is not a valid branch, tag, or commit"**  
Either the tag (`@v1`) doesn't exist on this repo (note: `@v1` does **not** auto‑resolve to `v1.0.0` — you have to create the `v1` alias explicitly, see *Versioning*), or access for other repos in the org hasn't been enabled (see *Access from other repos* above).

**Caller fails with `Secret X is required, but not provided while calling`**  
The reusable workflow declares `X` as `required: true` but the caller isn't forwarding it. Two common causes: (1) you used `secrets: inherit` but the caller and the template are under different owners — switch to explicit forwarding; (2) the secret isn't actually set on the caller repo — verify with `gh secret list --repo owner/repo`.

**Remote shell fails with `syntax error near unexpected token '('` (or similar)**  
Caused by passing values to `ssh user@host VAR=... 'bash -s'` unquoted — special characters in inputs like the commit message break the remote shell's parsing. The workflows in this repo avoid this by building the script with a quoted heredoc and piping it to ssh on stdin, with values exported via `printf '%q'`. If you fork the workflow, keep that pattern.

**Secrets show up as empty in the called workflow**  
The caller is missing `secrets: inherit` (or an explicit `secrets:` block listing each one). Reusable workflows don't automatically see the caller's secrets.

**Slack notification fires but message is empty**  
`SLACK_WEBHOOK_URL` isn't reaching the called workflow. Same fix as above.

---

## Roadmap

- Replace `sshpass` + password with SSH key auth via `webfactory/ssh-agent`.
- Add a `lint` reusable workflow (PHPStan / ESLint / Prettier).
- Add a `release` reusable workflow that tags + drafts GitHub releases.
