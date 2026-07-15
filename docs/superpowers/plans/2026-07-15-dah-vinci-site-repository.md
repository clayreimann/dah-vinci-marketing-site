# Dah Vinci Site Repository Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `/Users/clay/Code/claude/dah-vinci/dah-vinci-site` as a verified standalone Git repository and Cloudflare Workers Static Assets project, then remove website ownership from `morse-trainer` and clean its merge contents.

**Architecture:** Existing deployable website files remain under a dedicated `website/` asset directory in the new repository, while pinned Wrangler tooling and operator documentation live at the repository root and are never published as assets. The sibling repository is assembled, tested, and committed before the original website files are removed. The app and site coordinate only through stable public URLs and a release checklist.

**Tech Stack:** Static HTML/CSS/SVG, Node.js/npm, Cloudflare Wrangler, Workers Static Assets, Git, SwiftPM, XcodeGen/Xcode.

## Global Constraints

- The new site repository path is exactly `/Users/clay/Code/claude/dah-vinci/dah-vinci-site`.
- The future app path `/Users/clay/Code/claude/dah-vinci/dah-vinci-app` is not created or populated in this migration.
- The site repository uses `main` as its initial branch.
- Cloudflare Pages, Git submodules, and Git subtrees are not used.
- `morse-trainer/website` is not removed until the site repository has a verified initial commit.
- The website remains static-only: no Worker runtime entry point, API, analytics, or bindings.
- The pre-launch App Store CTA remains unchanged.
- The user-owned untracked file `docs/superpowers/plans/2026-07-15-pause-driven-keying.md` must remain untouched and must not enter migration commits.

---

### Task 1: Prepare website content and deployment scaffolding

**Files:**
- Modify: `/Users/clay/Code/claude/morse-trainer/website/privacy.html`
- Modify: `/Users/clay/Code/claude/morse-trainer/website/support.html`
- Replace: `/Users/clay/Code/claude/morse-trainer/website/DEPLOY.md`
- Create staging directory: `/tmp/dah-vinci-site-scaffold`
- Create: `/tmp/dah-vinci-site-scaffold/.gitignore`
- Create: `/tmp/dah-vinci-site-scaffold/README.md`
- Create: `/tmp/dah-vinci-site-scaffold/package.json`
- Create: `/tmp/dah-vinci-site-scaffold/package-lock.json`
- Create: `/tmp/dah-vinci-site-scaffold/wrangler.jsonc`

**Interfaces:**
- Consumes: Existing static pages and the stable support repository `https://github.com/clayreimann/morse-trainer`.
- Produces: Clean website content plus root-level deployment scaffolding that can be assembled into `dah-vinci-site` without exposing repository metadata as assets.

- [ ] **Step 1: Replace the public personal email with the issue tracker**

In both privacy and support pages, use this contact target:

```html
<a href="https://github.com/clayreimann/morse-trainer/issues">github.com/clayreimann/morse-trainer/issues</a>
```

The privacy sentence becomes:

```html
<p>Questions about this policy: open an issue at <a href="https://github.com/clayreimann/morse-trainer/issues">github.com/clayreimann/morse-trainer/issues</a>.</p>
```

The support contact row becomes:

```html
<div class="contact-row">
  <span class="signal"><i class="dot"></i><i class="dash"></i></span>
  <span>Bugs, questions, and suggestions: <a href="https://github.com/clayreimann/morse-trainer/issues">github.com/clayreimann/morse-trainer/issues</a></span>
</div>
```

- [ ] **Step 2: Add the site repository ignore policy**

Create `/tmp/dah-vinci-site-scaffold/.gitignore`:

```gitignore
node_modules/
.wrangler/
.DS_Store
```

- [ ] **Step 3: Add pinned Wrangler scripts**

Create `/tmp/dah-vinci-site-scaffold/package.json`:

```json
{
  "name": "dah-vinci-site",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "wrangler dev --local --port 8787",
    "check": "wrangler deploy --dry-run",
    "deploy": "wrangler deploy"
  },
  "devDependencies": {}
}
```

Run from `/tmp/dah-vinci-site-scaffold`:

```bash
npm install --save-dev --save-exact wrangler@latest
```

Expected: `package.json` gains an exact Wrangler version and `package-lock.json` is created.

- [ ] **Step 4: Add the static-only Worker configuration**

Create `/tmp/dah-vinci-site-scaffold/wrangler.jsonc`:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "dahvinci",
  "compatibility_date": "2026-07-15",
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

- [ ] **Step 5: Replace the deployment guide**

`website/DEPLOY.md` is prepared as the future root `DEPLOY.md` and must document these commands and settings:

```markdown
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

## First deployment

```sh
npx wrangler login
npx wrangler whoami
npm run deploy
```

The Worker is named `dahvinci`. Its custom domain is
`dahvinci.madtown.cloud`; Cloudflare creates the DNS record and certificate
from the custom-domain route in `wrangler.jsonc`.

## Workers Builds

Connect this repository to the existing Worker under **Settings → Builds**.

- Production branch: `main`
- Build command: none
- Production deploy command: `npm run deploy`
- Non-production deploy command: `npx wrangler versions upload`

## Rollback

Open the Worker in Cloudflare, select **Deployments**, choose the last
known-good version, and roll production back to that version. A failed build
does not replace the active deployment.
```

- [ ] **Step 6: Add the repository README**

Create `/tmp/dah-vinci-site-scaffold/README.md`:

```markdown
# Dah Vinci website

The public launch, support, and privacy website for Dah Vinci, a native Morse
code trainer. The app source lives in
[`clayreimann/morse-trainer`](https://github.com/clayreimann/morse-trainer).

## Public interfaces

- <https://dahvinci.madtown.cloud/>
- <https://dahvinci.madtown.cloud/privacy>
- <https://dahvinci.madtown.cloud/support>

These URLs are consumed by App Store metadata and must remain stable.

## App coordination

Update this repository when the app changes its privacy behavior, materially
changes its public feature or support descriptions, or publishes its App Store
listing. Routine app-only changes do not require a website deployment.

See [DEPLOY.md](DEPLOY.md) for local development, deployment, and rollback.
```

- [ ] **Step 7: Validate the prepared source**

Run:

```bash
cd /tmp/dah-vinci-site-scaffold
npm ci
```

Expected: dependency installation succeeds and the exact Wrangler dependency graph in `package-lock.json` is reproducible. Wrangler validation runs after the `website/` asset directory is assembled in Task 2.

### Task 2: Create and verify the independent site repository

**Files:**
- Create directory: `/Users/clay/Code/claude/dah-vinci/dah-vinci-site`
- Copy deployable files: `/Users/clay/Code/claude/morse-trainer/website/` to `/Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/`, excluding `DEPLOY.md`
- Copy scaffolding: `/tmp/dah-vinci-site-scaffold/` to the new repository root
- Copy deployment guide: `/Users/clay/Code/claude/morse-trainer/website/DEPLOY.md` to the new repository root
- Create: `/Users/clay/Code/claude/dah-vinci/dah-vinci-site/docs/superpowers/specs/2026-07-15-cloudflare-workers-deployment-design.md`
- Create: `/Users/clay/Code/claude/dah-vinci/dah-vinci-site/docs/superpowers/plans/2026-07-15-dah-vinci-site-repository.md`

**Interfaces:**
- Consumes: The complete source produced by Task 1.
- Produces: A standalone `main` Git repository whose initial commit is the deletion gate for `morse-trainer/website`.

- [ ] **Step 1: Create and initialize the destination**

Run:

```bash
mkdir -p /Users/clay/Code/claude/dah-vinci/dah-vinci-site
git init -b main /Users/clay/Code/claude/dah-vinci/dah-vinci-site
```

Expected: Git reports an empty repository on branch `main`.

- [ ] **Step 2: Copy the prepared website without deleting the source**

Run:

```bash
mkdir -p /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website
cp /Users/clay/Code/claude/morse-trainer/website/index.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/
cp /Users/clay/Code/claude/morse-trainer/website/privacy.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/
cp /Users/clay/Code/claude/morse-trainer/website/support.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/
cp /Users/clay/Code/claude/morse-trainer/website/404.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/
cp /Users/clay/Code/claude/morse-trainer/website/favicon.svg /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/
cp /Users/clay/Code/claude/morse-trainer/website/DEPLOY.md /Users/clay/Code/claude/dah-vinci/dah-vinci-site/DEPLOY.md
cp /tmp/dah-vinci-site-scaffold/.gitignore /Users/clay/Code/claude/dah-vinci/dah-vinci-site/.gitignore
cp /tmp/dah-vinci-site-scaffold/README.md /Users/clay/Code/claude/dah-vinci/dah-vinci-site/README.md
cp /tmp/dah-vinci-site-scaffold/package.json /Users/clay/Code/claude/dah-vinci/dah-vinci-site/package.json
cp /tmp/dah-vinci-site-scaffold/package-lock.json /Users/clay/Code/claude/dah-vinci/dah-vinci-site/package-lock.json
cp /tmp/dah-vinci-site-scaffold/wrangler.jsonc /Users/clay/Code/claude/dah-vinci/dah-vinci-site/wrangler.jsonc
mkdir -p /Users/clay/Code/claude/dah-vinci/dah-vinci-site/docs/superpowers/specs
mkdir -p /Users/clay/Code/claude/dah-vinci/dah-vinci-site/docs/superpowers/plans
cp /Users/clay/Code/claude/morse-trainer/docs/superpowers/specs/2026-07-15-cloudflare-workers-deployment-design.md /Users/clay/Code/claude/dah-vinci/dah-vinci-site/docs/superpowers/specs/
cp /Users/clay/Code/claude/morse-trainer/docs/superpowers/plans/2026-07-15-dah-vinci-site-repository.md /Users/clay/Code/claude/dah-vinci/dah-vinci-site/docs/superpowers/plans/
```

- [ ] **Step 3: Verify source/destination parity before any deletion**

Run:

```bash
diff -q /Users/clay/Code/claude/morse-trainer/website/index.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/index.html
diff -q /Users/clay/Code/claude/morse-trainer/website/privacy.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/privacy.html
diff -q /Users/clay/Code/claude/morse-trainer/website/support.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/support.html
diff -q /Users/clay/Code/claude/morse-trainer/website/404.html /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/404.html
diff -q /Users/clay/Code/claude/morse-trainer/website/favicon.svg /Users/clay/Code/claude/dah-vinci/dah-vinci-site/website/favicon.svg
diff -q /Users/clay/Code/claude/morse-trainer/website/DEPLOY.md /Users/clay/Code/claude/dah-vinci/dah-vinci-site/DEPLOY.md
```

Expected: all commands exit successfully with no output; none of the website or deployment-guide files differ.

Run from the destination repository:

```bash
npm ci
npm run check
```

Expected: dependency installation and Wrangler dry-run deployment validation both succeed without publishing.

- [ ] **Step 4: Verify local HTTP routing**

Start `npm run dev` in the site repository, then run:

```bash
curl --fail --silent --show-error http://127.0.0.1:8787/ >/dev/null
curl --fail --silent --show-error http://127.0.0.1:8787/privacy >/dev/null
curl --fail --silent --show-error http://127.0.0.1:8787/support >/dev/null
curl --silent --show-error --output /tmp/dah-vinci-404.html --write-out '%{http_code}' http://127.0.0.1:8787/not-a-real-page
```

Expected: the first three commands succeed and the last command prints `404`; `/tmp/dah-vinci-404.html` contains `Signal lost`.

- [ ] **Step 5: Verify repository hygiene**

Run:

```bash
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site status --short
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site check-ignore node_modules .wrangler .DS_Store
```

Expected: source files are untracked; ignored runtime directories are not candidates for the commit.

- [ ] **Step 6: Commit the destination repository**

Run:

```bash
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site add .
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site commit -m "feat: launch Dah Vinci website"
```

Expected: the initial commit succeeds on `main` and includes the static site, locked Wrangler setup, docs, spec, and plan.

- [ ] **Step 7: Confirm the deletion gate**

Run:

```bash
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site status --short --branch
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site ls-files
```

Expected: branch is `main`, the worktree is clean, and every intended site file is tracked.

### Task 3: Remove website ownership and stale artifacts from the app repository

**Files:**
- Delete: `/Users/clay/Code/claude/morse-trainer/website/`
- Delete: `/Users/clay/Code/claude/morse-trainer/IMPLEMENTATION.md`
- Delete: `/Users/clay/Code/claude/morse-trainer/docs/superpowers/specs/2026-07-15-cloudflare-workers-deployment-design.md`
- Delete: `/Users/clay/Code/claude/morse-trainer/docs/superpowers/plans/2026-07-15-dah-vinci-site-repository.md`
- Modify: `/Users/clay/Code/claude/morse-trainer/.gitignore`
- Modify: `/Users/clay/Code/claude/morse-trainer/docs/app-store/DISTRIBUTION.md`
- Modify: `/Users/clay/Code/claude/morse-trainer/docs/app-store/ExportOptions.plist`
- Replace: `/Users/clay/Code/claude/morse-trainer/docs/app-store/privacy-policy.md`
- Create: `/Users/clay/Code/claude/morse-trainer/docs/app-store/WEBSITE.md`

**Interfaces:**
- Consumes: The verified initial commit from Task 2 and stable URLs hosted by the site repository.
- Produces: An app-only repository ready for squash merge, with durable App Store documentation and explicit site coordination.

- [ ] **Step 1: Reconfirm the destination commit immediately before deletion**

Run:

```bash
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site log -1 --oneline
git -C /Users/clay/Code/claude/dah-vinci/dah-vinci-site status --short
```

Expected: one initial commit exists and status is empty. Do not continue otherwise.

- [ ] **Step 2: Remove migrated and consumed files**

Run only after Step 1 passes:

```bash
git rm -r website IMPLEMENTATION.md \
  docs/superpowers/specs/2026-07-15-cloudflare-workers-deployment-design.md \
  docs/superpowers/plans/2026-07-15-dah-vinci-site-repository.md
```

Do not remove or unstage any pause-driven-keying design or plan files.

- [ ] **Step 3: Ignore the local redesign PDF**

Append to the app `.gitignore`:

```gitignore

# Local design reference
Morse redesign.pdf
```

- [ ] **Step 4: Make distribution documentation durable**

Replace `docs/app-store/DISTRIBUTION.md` with:

````markdown
# Distribution guide — TestFlight / App Store

## Prerequisites

1. Join the paid Apple Developer Program for the team that will publish the app.
2. Create the App Store Connect app record for bundle ID `cloud.madtown.morse.app`.
3. Put `DEVELOPMENT_TEAM` and `CODE_SIGN_STYLE = Automatic` in the ignored
   `Signing.xcconfig`, using the publishing team identifier shown in the Apple
   Developer membership page.

## Xcode Organizer

1. Run `xcodegen generate` and open `MorseTrainerApp.xcodeproj`.
2. Select `MorseTrainerApp-iOS` and **Any iOS Device**.
3. Confirm automatic signing uses the publishing team.
4. Choose **Product → Archive**.
5. In Organizer, choose **Distribute App → App Store Connect → Upload**.
6. After processing, attach the build to TestFlight or the App Store version.

## Command line

```sh
xcodebuild archive \
  -project MorseTrainerApp.xcodeproj \
  -scheme MorseTrainerApp-iOS \
  -destination 'generic/platform=iOS' \
  -archivePath build/MorseTrainer.xcarchive \
  -allowProvisioningUpdates

xcodebuild -exportArchive \
  -archivePath build/MorseTrainer.xcarchive \
  -exportOptionsPlist docs/app-store/ExportOptions.plist \
  -exportPath build/export \
  -allowProvisioningUpdates
```

Upload the exported app with Xcode Organizer or Transporter. Never commit App
Store Connect API keys, `.p8` files, signing certificates, or provisioning
profiles.

The App Store copy and URLs are maintained in [metadata.md](metadata.md).
````

- [ ] **Step 5: Remove the tracked Team ID**

Delete the `teamID` key/value and its account-specific comment from `docs/app-store/ExportOptions.plist`. Preserve `method`, `signingStyle`, `destination`, and `uploadSymbols`.

- [ ] **Step 6: Replace duplicate privacy policy ownership**

Replace `docs/app-store/privacy-policy.md` with:

```markdown
# Privacy Policy — Dah Vinci

The canonical, publishable privacy policy is maintained in the
[`dah-vinci-site`](https://github.com/clayreimann/dah-vinci-site) repository and
served at <https://dahvinci.madtown.cloud/privacy>.

Any app change affecting data collection, storage, transmission, tracking, or
required-reason APIs must update the site policy before the app release is
submitted.
```

- [ ] **Step 7: Add the release coordination note**

Create `docs/app-store/WEBSITE.md`:

```markdown
# Website release coordination

The public site is maintained separately in
[`clayreimann/dah-vinci-site`](https://github.com/clayreimann/dah-vinci-site).

Stable App Store URLs:

- Marketing: <https://dahvinci.madtown.cloud/>
- Privacy: <https://dahvinci.madtown.cloud/privacy>
- Support: <https://dahvinci.madtown.cloud/support>

Before an app release:

- Privacy behavior changed: update and deploy the site policy first.
- Material public features or support instructions changed: update the site.
- App Store listing became public: replace the pre-launch CTA with its URL.
- Confirm `/`, `/privacy`, and `/support` return successful HTTPS responses before submission.
```

- [ ] **Step 8: Verify app repository hygiene and behavior**

Run:

```bash
git -C /Users/clay/Code/claude/morse-trainer diff --check
git -C /Users/clay/Code/claude/morse-trainer diff --cached --check
swift test
xcodegen generate
xcodebuild -quiet -project MorseTrainerApp.xcodeproj -scheme MorseTrainerApp-iOS -configuration Debug -destination 'generic/platform=iOS Simulator' -derivedDataPath /tmp/morse-trainer-derived-data CODE_SIGNING_ALLOWED=NO build
plutil -extract UILaunchScreen xml1 -o - /tmp/morse-trainer-derived-data/Build/Products/Debug-iphonesimulator/MorseTrainerApp.app/Info.plist
```

Expected: no whitespace errors, 44 Swift tests pass, the iOS build succeeds, and plist extraction returns a dictionary.

- [ ] **Step 9: Commit only migration-owned app files**

Stage and commit only the migration paths so unrelated user-owned planning files remain outside the commit:

```bash
git add -A -- .gitignore IMPLEMENTATION.md website \
  docs/superpowers/specs/2026-07-15-cloudflare-workers-deployment-design.md \
  docs/superpowers/plans/2026-07-15-dah-vinci-site-repository.md \
  docs/app-store/DISTRIBUTION.md \
  docs/app-store/ExportOptions.plist \
  docs/app-store/privacy-policy.md \
  docs/app-store/WEBSITE.md

git commit --only -m "chore: extract website into standalone repository" -- \
  .gitignore IMPLEMENTATION.md website \
  docs/superpowers/specs/2026-07-15-cloudflare-workers-deployment-design.md \
  docs/superpowers/plans/2026-07-15-dah-vinci-site-repository.md \
  docs/app-store/DISTRIBUTION.md \
  docs/app-store/ExportOptions.plist \
  docs/app-store/privacy-policy.md \
  docs/app-store/WEBSITE.md
```

### Task 4: Publish the first Worker deployment

**Files:**
- Read: `/Users/clay/Code/claude/dah-vinci/dah-vinci-site/wrangler.jsonc`
- Read: `/Users/clay/Code/claude/dah-vinci/dah-vinci-site/DEPLOY.md`

**Interfaces:**
- Consumes: The clean, committed site repository from Task 2.
- Produces: A Cloudflare Worker named `dahvinci` serving the static site at its `workers.dev` hostname and `dahvinci.madtown.cloud`.

- [ ] **Step 1: Confirm Wrangler authentication**

Run:

```bash
cd /Users/clay/Code/claude/dah-vinci/dah-vinci-site
npx wrangler whoami
```

Expected: the intended Cloudflare account is displayed. If not authenticated, run `npx wrangler login` and complete the browser authorization.

- [ ] **Step 2: Deploy production**

Run:

```bash
npm run deploy
```

Expected: Wrangler uploads the static assets, deploys Worker `dahvinci`, and reports a deployment/version identifier and URL.

- [ ] **Step 3: Verify public routing and DNS**

Run:

```bash
curl --fail --silent --show-error https://dahvinci.madtown.cloud/ >/dev/null
curl --fail --silent --show-error https://dahvinci.madtown.cloud/privacy >/dev/null
curl --fail --silent --show-error https://dahvinci.madtown.cloud/support >/dev/null
curl --silent --show-error --output /tmp/dah-vinci-public-404.html --write-out '%{http_code}' https://dahvinci.madtown.cloud/not-a-real-page
```

Expected: the first three commands succeed and the final command prints `404` with the custom page body.

- [ ] **Step 4: Record the GitHub/Workers Builds handoff**

Do not create a GitHub repository without explicit confirmation. Report that the local site repository is ready for `gh repo create clayreimann/dah-vinci-site --public --source . --remote origin --push`, after which the existing Worker can be connected under **Settings → Builds** with `main` as production, no build command, `npm run deploy` as production deploy, and `npx wrangler versions upload` for non-production branches.

### Task 5: Refresh the app pull request record

**Files:**
- External metadata: `https://github.com/clayreimann/morse-trainer/pull/1`

**Interfaces:**
- Consumes: Final app repository diff and fresh verification results.
- Produces: An accurate permanent PR summary for the squash merge.

- [ ] **Step 1: Re-read the final branch diff and verification evidence**

Run:

```bash
git -C /Users/clay/Code/claude/morse-trainer diff --stat main...HEAD
git -C /Users/clay/Code/claude/morse-trainer log --oneline main..HEAD
```

- [ ] **Step 2: Update pull request 1**

The PR body must describe the current iOS/macOS app, 44-test Swift suite, App Store readiness, successful iOS simulator build and `UILaunchScreen` verification, and that the website was extracted to `dah-vinci-site`. It must not claim the site remains in this repository or reference removed `TimingMeter` UI.

- [ ] **Step 3: Final status audit**

Confirm both repositories' branch/status, list the site commit and app cleanup commit, and verify the untracked pause-driven-keying plan remains untouched in `morse-trainer`.
