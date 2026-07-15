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
