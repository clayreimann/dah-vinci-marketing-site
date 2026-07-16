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
- Preview URLs: public

Every non-production branch push validates the configuration and uploads a
Worker version without changing production. When the branch has an open pull
request, Cloudflare reports the build and public preview URL on the PR.

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
`200`, and that an unknown path returns the custom `404` page.

## Rollback

Open the Worker in Cloudflare, select **Deployments**, choose the last
known-good version, and roll production back to that version. A failed build or
preview upload does not replace the active deployment. After rollback, fix the
repository through the normal pull-request flow.
