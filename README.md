# AI Infra Wiki

Quartz site for the AI infrastructure papers selected from OSDI 2026.

## Local preview

```sh
cd quartz
npm ci
npx quartz build --serve -d ../wiki
```

## Cloudflare Pages

Connect this repository to Cloudflare Pages and set the production branch to `ai-infra`, root directory to `quartz`, build command to `npm ci && npx quartz build -d ../wiki`, and output directory to `public`. Set `NODE_VERSION=22` and `BASE_URL` to the site's `*.pages.dev` hostname.
