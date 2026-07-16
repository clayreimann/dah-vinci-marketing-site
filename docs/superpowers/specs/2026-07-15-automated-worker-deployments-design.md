# Automated Cloudflare Worker Deployments Design

**Goal:** Make pull requests produce public Cloudflare Worker previews and make
merges to protected `main` deploy the Dah Vinci marketing site to production.

## Decisions

- Use Cloudflare Workers Builds with its native GitHub integration. Do not add
  a GitHub Actions deployment workflow or store Cloudflare credentials in
  GitHub.
- Connect the existing `dahvinci` Worker to
  `clayreimann/dah-vinci-marketing-site`; do not create a second Worker or a
  Cloudflare Pages project.
- Keep `main` as the only production branch.
- Enable builds for non-production branches and use public Worker preview URLs.
  The source repository is public and preview content is not embargoed, so
  Cloudflare Access would add friction without protecting private material.
- Protect `main` with a GitHub ruleset that requires a pull request and the
  exact `Workers Builds: dahvinci` check from the **Cloudflare Workers and
  Pages** GitHub App (`cloudflare-workers-and-pages`), but requires zero human
  approvals. This supports a solo maintainer without allowing unverified direct
  pushes or a same-named check from another integration.
- Keep Wrangler configuration and deployment documentation in the repository
  as the durable source of truth.

## Repository configuration

Add `preview_urls: true` to `wrangler.jsonc`. This makes preview availability
explicit and prevents a later dashboard deployment from silently reverting the
setting to a default derived from `workers_dev`.

Keep the existing package scripts:

- `npm run check` performs a Wrangler dry run and validates the asset manifest
  and configuration.
- `npm run deploy` creates a Worker version and promotes it to production.
- Local development remains `npm run dev`.

Update `DEPLOY.md` to describe the automated branch behavior, public preview
URLs, GitHub protection, failure handling, and manual rollback.

## Cloudflare Workers Builds settings

Connect the existing Worker through **Workers & Pages → dahvinci → Settings →
Builds** with these values:

- Git repository: `clayreimann/dah-vinci-marketing-site`
- Production branch: `main`
- Root directory: repository root
- Build command: `npm run check`
- Production deploy command: `npm run deploy`
- Non-production branch deploy command: `npx wrangler versions upload`
- Builds for non-production branches: enabled
- Preview URLs: enabled and public
- Ordinary production `workers.dev` route: disabled

Cloudflare Workers Builds manages the deployment credential. No Cloudflare API
token is committed to the repository or added as a GitHub Actions secret.

## GitHub protection

Create an active repository ruleset named `Protect main` targeting exactly
`refs/heads/main` after Cloudflare has reported its first check. Configure it
to:

- Require changes to arrive through a pull request.
- Require zero approving reviews.
- Require the exact `Workers Builds: dahvinci` check from the **Cloudflare
  Workers and Pages** GitHub App (`cloudflare-workers-and-pages`).
- Require the pull request branch to be up to date with `main` before merging.
- Block force pushes.
- Block deletion of `main`.

Do not add a routine bypass. If Cloudflare has an extended outage during an
urgent production incident, an administrator may temporarily edit the ruleset;
that exceptional change remains visible in repository administration history.

## Deployment data flow

### Pull request preview

1. A contributor pushes a non-production branch.
2. Workers Builds installs the locked dependencies and runs `npm run check`.
3. If validation succeeds, Workers Builds runs
   `npx wrangler versions upload`.
4. Cloudflare creates a new Worker version without promoting it to production.
5. Cloudflare publishes a public version and branch preview URL, a GitHub check
   run, and a pull-request comment when the branch has an open PR. The check and
   comment confirm build status, but the comment is not required to contain the
   preview link. Cloudflare's dashboard and Worker version metadata are the
   source of truth for the preview alias.
6. Review verifies `/`, `/privacy`, `/support`, and an unknown route on the
   preview before merge.

For this Worker, the stable branch preview format is
`https://<sanitized-branch>-dahvinci.c-jensenreimann.workers.dev`. Preview URLs
must be enabled with Public visibility while the ordinary production
`dahvinci.c-jensenreimann.workers.dev` route remains disabled.

### Production deployment

1. GitHub permits the PR to merge only after the required Cloudflare check
   succeeds and the branch is current with `main`.
2. The merge commit on `main` triggers another Workers Build.
3. Workers Builds runs `npm run check`, then `npm run deploy`.
4. The new Worker version becomes the active deployment for
   `dahvinci.madtown.cloud`.
5. Post-deploy checks verify the three stable public routes and custom 404.

Direct pushes to `main` are rejected by the GitHub ruleset, so production
changes follow the same preview-and-review path.

## Failure handling and rollback

- A failed validation or preview upload reports a failed GitHub check and
  blocks the PR from merging.
- A failed production build does not replace the active Worker version.
- A successful preview version is never promoted merely because it exists;
  only the production branch deploy command changes active traffic.
- Roll production back from the Worker's deployment history to the last
  known-good version, then fix the repository through the normal pull-request
  path so declared source and deployed state converge again.
- Preview URLs are disposable Cloudflare version endpoints. No custom preview
  DNS records, cleanup workflow, or preview secrets are required.

## Rollout sequence

1. Create a feature branch containing `preview_urls: true` and the updated
   deployment documentation.
2. Connect the existing `dahvinci` Worker to the GitHub repository and enable
   non-production branch builds.
3. Push the feature branch after the Git integration is connected so the push
   triggers its first non-production Workers Build.
4. Open a pull request from the feature branch so Cloudflare reports the
   preview, build-status PR comment, and named check run.
5. Verify preview routing and content.
6. Create the active `Protect main` ruleset targeting exactly
   `refs/heads/main`, and select the observed Cloudflare check as required.
7. Merge the pull request.
8. Verify the resulting production build and the live custom-domain routes.

This order avoids configuring a required status check before GitHub has seen
that check name and source.

## Verification

- `npm ci` succeeds from the committed lockfile.
- `npm run check` succeeds locally and in Workers Builds.
- A non-production branch creates a Worker version but does not change the
  active production version.
- The pull request receives a successful `Workers Builds: dahvinci` check from
  the **Cloudflare Workers and Pages** GitHub App
  (`cloudflare-workers-and-pages`), and its public branch alias is discoverable
  from Cloudflare's dashboard or Worker version metadata even if the
  build-status comment omits the URL.
- Preview `/`, `/privacy`, and `/support` return `200`; an unknown path returns
  the custom `404` page.
- GitHub rejects a direct update to protected `main` and prevents merging while
  the Cloudflare check is pending or failed.
- Merging a passing PR triggers a production deployment.
- Production routes return `200`, `200`, `200`, and `404` respectively after
  the merge.
- No Cloudflare API token or GitHub Actions deployment secret exists in the
  repository.

## Non-goals

- No staging custom domain.
- No Cloudflare Access policy for previews.
- No per-PR Worker names, DNS records, runtime bindings, or data stores.
- No GitHub Actions deployment workflow.
- No automated promotion of an arbitrary preview version outside a merge to
  `main`.
