# AI Infra Wiki

Quartz site for 49 AI infrastructure papers selected from OSDI 2026 and 18 user-selected papers from SOSP 2026. The SOSP collection includes 11 full-text notes and 7 clearly marked abstract-only notes.

## Local preview

cd quartz
npm ci
npx quartz build --serve -d ../wiki

## Cloudflare Pages

Production branch: ai-infra
Root directory: quartz
Build command: npm ci && npx quartz build -d ../wiki
Build output directory: public
Site URL: https://ai-infra-wiki-erx.pages.dev

Set BASE_URL if you later use a custom domain.
