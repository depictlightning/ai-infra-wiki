# AI Infra Wiki

Quartz site for the AI infrastructure papers selected from OSDI 2026.

## Local preview

```sh
cd quartz
npm ci
npx quartz build --serve -d ../wiki
```

## Cloudflare Workers

Connect the `ai-infra` branch to a Worker named `ai-infra-wiki`. In Advanced settings, set the root directory to `quartz`. Use `npm ci && npx quartz build -d ../wiki` as the build command and `npx wrangler deploy` as the deploy command. Set `NODE_VERSION=22` and `BASE_URL` to the Worker hostname without `https://` once Cloudflare assigns it. The static assets in `quartz/public` are deployed using `quartz/wrangler.jsonc`.
