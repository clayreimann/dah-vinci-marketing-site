# Dah Vinci Site Repository and Cloudflare Deployment Design

**Goal:** Extract the Dah Vinci launch website into an independent, repository-owned Cloudflare Workers Static Assets project at `/Users/clay/Code/claude/dah-vinci/dah-vinci-site`, while preserving the current site until the new repository is complete and verified.

## Decisions

- Create a new product folder at `/Users/clay/Code/claude/dah-vinci` alongside the current `/Users/clay/Code/claude/morse-trainer` checkout.
- Create an independent Git repository at `/Users/clay/Code/claude/dah-vinci/dah-vinci-site`.
- Eventually move the app into `/Users/clay/Code/claude/dah-vinci/dah-vinci-app`, but do not relocate the app during this migration.
- Use Cloudflare Workers Static Assets. Do not create a Cloudflare Pages project.
- Keep all website deploy behavior in the site repository so a fresh clone contains the complete deployment contract.
- Use `dahvinci.madtown.cloud` as a Worker Custom Domain. Cloudflare owns DNS and certificate provisioning for the hostname.
- Keep the current pre-launch “Coming to the App Store” copy until the App Store listing is public.
- Use ordinary repositories rather than a Git submodule or subtree.

## Site repository contents

The root of `dah-vinci-site` contains:

- A `website/` directory containing only the deployable `index.html`, `privacy.html`, `support.html`, `404.html`, and `favicon.svg` files.
- `wrangler.jsonc` as the deployment source of truth. It names the Worker `dahvinci`, points static assets at `website/`, enables the existing `404.html`, serves `.html` files at extensionless URLs, and declares the custom domain.
- `package.json` with local preview, validation, and deployment scripts.
- `package-lock.json` pinning the tested Wrangler dependency graph for local and Cloudflare builds.
- `.gitignore` excluding `node_modules/`, Wrangler state, and operating-system metadata.
- `README.md` describing the site, local development, and its coordination contract with the app.
- `DEPLOY.md` documenting first deployment, Workers Builds Git integration, validation, and rollback.
- The Cloudflare deployment design spec, moved from the app repository after it has served as the migration plan.

## Ordered migration

The migration must preserve the source material until the destination is safe:

1. Create `/Users/clay/Code/claude/dah-vinci/dah-vinci-site` and initialize it as a Git repository with `main` as its initial branch.
2. Copy the deployable files from `morse-trainer/website/` into the destination `website/` directory, move deployment documentation to the destination repository root, and copy the deployment design spec into the new repository.
3. Add the site repository configuration and documentation.
4. Verify file parity, local routing, Wrangler configuration, and Git status inside the new repository.
5. Commit the complete initial site repository.
6. Only after that commit succeeds, remove `website/` and the website deployment spec from `morse-trainer`.
7. Complete the remaining app-repository cleanup and verify the app repository independently.

If any destination setup or verification step fails, leave the existing `morse-trainer/website/` directory untouched.

## App repository cleanup

- Remove the consumed root `IMPLEMENTATION.md` handoff.
- Keep `Morse redesign.pdf` untracked and explicitly ignored.
- Rewrite the App Store distribution guide as durable operator documentation: remove transient certificate observations, assistant-specific prose, and the incorrect `notarytool` upload suggestion.
- Remove the account-specific Team ID from the tracked export options. Local signing remains in the ignored `Signing.xcconfig`; exported archives use their existing signing team.
- Replace the public personal Gmail address in the migrated website with the app repository issue tracker as the support and privacy contact.
- Keep App Store metadata and the privacy manifest in the app repository. Replace any duplicate publishable privacy-policy source with a link to the canonical site repository or deployed policy.
- Add a short release-coordination note linking the app repository to the site repository.
- Update pull request 1 so its permanent merge record reflects the current app, test count, verification status, and the extraction of the website.

## Cross-repository coordination contract

- `https://dahvinci.madtown.cloud`, `/privacy`, and `/support` are stable public interfaces consumed by App Store metadata.
- A change to app data collection or privacy behavior requires the site privacy policy to be updated before the app release is submitted.
- A material change to public features or support instructions requires a site update as part of the app release checklist.
- Publishing the App Store listing requires a site change replacing the pre-launch CTA with the final listing URL.
- Routine app-only changes do not trigger or require a website deployment.

## Deployment flow

1. Install the locked Node dependencies with `npm ci` after the initial lockfile is created.
2. Validate the Worker configuration without publishing.
3. Preview the static site locally and verify `/`, `/privacy`, `/support`, and an unknown route.
4. Authenticate Wrangler to the intended Cloudflare account.
5. Deploy the Worker and static assets. The custom-domain declaration provisions `dahvinci.madtown.cloud` in the existing `madtown.cloud` Cloudflare zone.
6. Verify both the generated `workers.dev` hostname and custom domain before using the URLs in App Store Connect.
7. Connect the existing Worker to the `dah-vinci-site` GitHub repository through Workers Builds. Production deploys come from `main`; non-production branches upload preview versions rather than replacing production.

## Failure handling and rollback

- Configuration validation and local preview happen before the first publish.
- A failed Wrangler or Workers Build command stops the deployment with a nonzero status.
- The existing active Worker deployment remains available if a later build fails.
- Use Cloudflare version history to roll production back to the previous known-good version.
- Do not add Cloudflare API tokens to either repository. Local deployment uses Wrangler login; Workers Builds manages its Cloudflare credentials internally.
- Do not delete the original website files until the destination repository has a verified initial commit.

## Verification

### Site repository

- Every file from `morse-trainer/website/` exists in the new repository before source removal.
- `npm ci` succeeds from a clean dependency state.
- Wrangler configuration validation succeeds.
- Local HTTP checks return `200` for `/`, `/privacy`, and `/support`, and `404` with the custom page for an unknown route.
- The initial site commit contains no dependency directory, Wrangler state, or macOS metadata.
- The first Cloudflare deployment completes successfully.
- The custom hostname resolves and serves the same content over HTTPS.

### App repository

- `website/`, the consumed implementation handoff, and the website deployment spec are absent only after the site repository commit succeeds.
- App Store URLs still target the stable custom domain.
- `swift test` and the unsigned iOS simulator build continue to pass.
- The built iOS app still contains the `UILaunchScreen` key.
- `git diff --check` reports no whitespace errors.

## Non-goals

- No Pages project or Pages Functions.
- No server-side Worker code, API endpoints, analytics, or runtime bindings.
- No relocation or renaming of the app repository during this migration.
- No Git submodule or subtree relationship between the repositories.
- No launch-day replacement of the App Store CTA before the listing URL exists.
