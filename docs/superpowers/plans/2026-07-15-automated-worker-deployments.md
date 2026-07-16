# Automated Cloudflare Worker Deployments Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every marketing-site pull request a public Cloudflare Worker preview and deploy every protected `main` merge to `dahvinci.madtown.cloud`.

**Architecture:** Cloudflare Workers Builds connects the existing `dahvinci` Worker directly to the public GitHub repository. Non-production branches upload immutable Worker versions for preview; only `main` runs the production deploy command. A GitHub ruleset requires the Cloudflare check before merge and prevents direct or force pushes to production.

**Tech Stack:** Cloudflare Workers Static Assets, Wrangler 4.111.0, Workers Builds GitHub integration, GitHub repository rulesets, npm.

## Global Constraints

- Use the existing Worker named `dahvinci`; do not create a Worker or Pages project.
- Production branch is exactly `main`.
- Preview URLs are public and explicitly enabled with `preview_urls: true`.
- Use Cloudflare Workers Builds; do not add GitHub Actions or GitHub-hosted Cloudflare credentials.
- Require pull requests and the Cloudflare check on `main`, with zero human approvals.
- Require protected branches to be current before merge; block force pushes and deletion of `main`.
- Stable production routes remain `https://dahvinci.madtown.cloud/`, `/privacy`, and `/support`.

---

### Task 1: Declare preview behavior and automated deployment operations

**Files:**
- Modify: `wrangler.jsonc`
- Modify: `DEPLOY.md`

**Interfaces:**
- Consumes: Existing Worker name `dahvinci`, asset directory `website/`, custom domain `dahvinci.madtown.cloud`, and npm scripts `check` and `deploy`.
- Produces: Repository-declared preview URL behavior and the operator contract used to configure Workers Builds and GitHub protection.

- [ ] **Step 1: Verify the current config has no explicit preview policy**

Run:

```bash
rg -n 'preview_urls' wrangler.jsonc
```

Expected: exit 1 with no matches.

- [ ] **Step 2: Add the explicit preview URL policy**

Modify `wrangler.jsonc` to contain exactly:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "dahvinci",
  "compatibility_date": "2026-07-15",
  "preview_urls": true,
  "assets": {
    "directory": "./website",
    "not_found_handling": "404-page",
    "html_handling": "drop-trailing-slash"
  },
  "routes": [
    {
      "pattern": "dahvinci.madtown.cloud",
      "custom_domain": true
    }
  ]
}
```

- [ ] **Step 3: Replace the manual-only Workers Builds section with the automated operating contract**

Replace `DEPLOY.md` with:

````markdown
# Deploying the Dah Vinci website

This repository deploys a static site to Cloudflare Workers Static Assets. The
deployable files live in `website/`; `wrangler.jsonc` is the deployment source
of truth.

## Local setup and validation

```sh
npm ci
npm run check
npm run dev
```

Wrangler serves the local site at `http://127.0.0.1:8787`. Verify `/`,
`/privacy`, `/support`, and an unknown path before publishing.

## Automated deployment flow

The existing `dahvinci` Worker is connected to
`clayreimann/dah-vinci-marketing-site` through Cloudflare Workers Builds.

- Production branch: `main`
- Build command: `npm run check`
- Production deploy command: `npm run deploy`
- Non-production deploy command: `npx wrangler versions upload`
- Non-production branch builds: enabled
- Preview URLs: enabled and public
- Ordinary production `workers.dev` route: disabled

Every non-production branch push validates the configuration and uploads a
Worker version without changing production. When the branch has an open pull
request, Cloudflare's GitHub check and comment confirm the build result. The
comment may omit the preview link even when the preview is healthy. The stable
branch URL is
`https://<sanitized-branch>-dahvinci.c-jensenreimann.workers.dev`; Cloudflare's
dashboard and Worker version metadata are the source of truth for the alias.
For example, `codex/automated-worker-deployments` becomes
`codex-automated-worker-deployments` in that hostname.

`main` is protected by a GitHub ruleset. Changes require a pull request with a
successful Cloudflare build; no human approval is required. Merging a passing
PR triggers a production build and deploys the new version to
`dahvinci.madtown.cloud`.

## Manual deployment

Use a manual deployment only when repairing the automated integration:

```sh
npx wrangler login
npx wrangler whoami
npm run deploy
```

After a manual deployment, reconcile the same source and configuration through
the normal pull-request flow so Git remains the deployment source of truth.

## Verification

For previews and production, verify that `/`, `/privacy`, and `/support` return
`200`, and that an unknown path returns the custom `404` page. Before testing a
preview, confirm in Cloudflare that **Preview URLs** are enabled with Public
visibility while the ordinary production `workers.dev` route remains disabled.

## Rollback

Open the Worker in Cloudflare, select **Deployments**, choose the last
known-good version, and roll production back to that version. A failed build or
preview upload does not replace the active deployment. After rollback, fix the
repository through the normal pull-request flow.
````

- [ ] **Step 4: Install the locked dependencies and validate the changed config**

Run:

```bash
npm ci
npm run check
git diff --check
```

Expected: npm reports zero vulnerabilities; Wrangler reads five assets, reports
no bindings, and exits after a dry run; `git diff --check` emits no output.

- [ ] **Step 5: Commit the repository configuration**

Run:

```bash
git add wrangler.jsonc DEPLOY.md
git commit -m "chore: automate Worker deployments"
```

Expected: one commit containing only `wrangler.jsonc` and `DEPLOY.md`.

---

### Task 2: Connect Workers Builds and prove the preview path

**Files:**
- None

**Interfaces:**
- Consumes: Worker `dahvinci`, repository `clayreimann/dah-vinci-marketing-site`, branch `codex/automated-worker-deployments`, and the commands declared in Task 1.
- Produces: A GitHub-connected Workers Build, a PR check, a PR comment, and public version/branch preview URLs without changing production traffic.

- [ ] **Step 1: Record the active production deployment before connecting Git**

Run:

```bash
npx wrangler deployments status
```

Expected: production status identifies the currently active `dahvinci` version.
Save that version ID in the execution notes for comparison after the preview.

- [ ] **Step 2: Connect the existing Worker to GitHub**

In Cloudflare, open **Workers & Pages → dahvinci → Settings → Builds →
Connect**. Authorize the **Cloudflare Workers and Pages** GitHub App for only
`clayreimann/dah-vinci-marketing-site`, then save these settings:

```text
Git repository: clayreimann/dah-vinci-marketing-site
Production branch: main
Root directory: /
Build command: npm run check
Deploy command: npm run deploy
Non-production branch deploy command: npx wrangler versions upload
Builds for non-production branches: enabled
Preview URLs: enabled with Public visibility
Ordinary production workers.dev route: disabled
```

Expected: the Worker remains named `dahvinci`, Builds shows the selected GitHub
repository, and no second Worker is created.

- [ ] **Step 3: Push the feature branch after the integration is connected**

Run:

```bash
git push -u origin codex/automated-worker-deployments
```

Expected: GitHub creates the remote feature branch and Workers Builds starts a
non-production build for its head commit.

- [ ] **Step 4: Open the rollout pull request**

Run:

```bash
gh pr create \
  --base main \
  --head codex/automated-worker-deployments \
  --title "Automate Cloudflare preview and production deploys" \
  --body $'Connect the marketing site to Cloudflare Workers Builds, publish public previews for non-production branches, document automatic production deploys, and prepare main-branch protection.\n\nValidation: npm ci; npm run check; git diff --check.'
```

Expected: GitHub creates one open PR targeting `main`.

- [ ] **Step 5: Wait for and inspect the Cloudflare check**

Run:

```bash
gh pr checks --watch
gh pr view --comments
```

Expected: the Cloudflare check succeeds and its PR comment confirms the build
result. The comment may omit the preview URL. Record the exact check name shown
by GitHub; Task 3 uses that observed name in the ruleset. Confirm in Cloudflare
that Preview URLs are enabled with Public visibility while the ordinary
production `workers.dev` route is disabled.

- [ ] **Step 6: Verify the public preview routes**

Use Cloudflare's dashboard or Worker version metadata to find the stable branch
alias. For this Worker its format is
`https://<sanitized-branch>-dahvinci.c-jensenreimann.workers.dev`. Then run:

```bash
read "PREVIEW_URL?Paste the Cloudflare preview URL: "
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "$PREVIEW_URL/"
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "$PREVIEW_URL/privacy"
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "$PREVIEW_URL/support"
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "$PREVIEW_URL/not-a-real-page"
```

Expected, in order: `200`, `200`, `200`, `404`.

- [ ] **Step 7: Prove the preview did not replace production**

Run:

```bash
npx wrangler deployments status
```

Expected: the active production version ID is unchanged from Step 1, while the
preview version appears in Cloudflare version history.

---

### Task 3: Protect main, merge, and verify production automation

**Files:**
- None

**Interfaces:**
- Consumes: The successful Cloudflare check name and GitHub App source observed in Task 2.
- Produces: An active default-branch ruleset and a verified production deploy triggered only by merging a passing PR.

- [ ] **Step 1: Create the active `main` ruleset after the Cloudflare check exists**

In GitHub, open **Settings → Rules → Rulesets → New branch ruleset** and enter:

```text
Ruleset name: Protect production main
Enforcement status: Active
Target branches: Include default branch
Bypass list: empty
Restrict deletions: enabled
Require a pull request before merging: enabled
Required approvals: 0
Require status checks to pass: enabled
Required check: the Cloudflare check observed in Task 2, sourced from the Cloudflare GitHub App
Require branches to be up to date before merging: enabled
Block force pushes: enabled
```

Expected: GitHub creates one active repository ruleset targeting `main`.

- [ ] **Step 2: Verify the ruleset through the GitHub API**

Run:

```bash
gh api repos/clayreimann/dah-vinci-marketing-site/rulesets \
  --jq '.[] | select(.name == "Protect production main") | {name, enforcement, target}'
```

Expected:

```json
{"enforcement":"active","name":"Protect production main","target":"branch"}
```

- [ ] **Step 3: Confirm the PR is mergeable under the new rule**

Run:

```bash
gh pr checks
gh pr view --json mergeStateStatus,reviewDecision,statusCheckRollup
```

Expected: Cloudflare is successful, no review approval is required, and the PR
is mergeable. If GitHub reports the branch is behind, update the branch and
wait for the Cloudflare check to rerun before continuing.

- [ ] **Step 4: Squash merge the rollout PR**

Run:

```bash
gh pr merge --squash --delete-branch
```

Expected: GitHub merges the PR into protected `main` and deletes the remote
feature branch. The merge triggers a production Workers Build.

- [ ] **Step 5: Wait for the production build and inspect the active version**

In Cloudflare, open **Workers & Pages → dahvinci → Deployments → Build
history** and wait for the `main` build to succeed. Then run:

```bash
npx wrangler deployments status
```

Expected: the active production version differs from Task 2 Step 1 and reports
the Git-triggered deployment.

- [ ] **Step 6: Verify production routes after the automated deploy**

Run:

```bash
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "https://dahvinci.madtown.cloud/"
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "https://dahvinci.madtown.cloud/privacy"
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "https://dahvinci.madtown.cloud/support"
curl --silent --show-error --output /dev/null --write-out "%{http_code}\n" "https://dahvinci.madtown.cloud/not-a-real-page"
```

Expected, in order: `200`, `200`, `200`, `404`.

- [ ] **Step 7: Synchronize the local checkout and perform the final audit**

Run:

```bash
git switch main
git pull --ff-only
git status -sb
git log -1 --oneline
gh pr view --json state,mergedAt,url
```

Expected: local `main` tracks `origin/main` with no changes, the rollout PR is
merged, and the latest commit is the squash merge for deployment automation.
