# AI Infra Wiki

Quartz site for 49 AI infrastructure papers selected from OSDI 2026, 18 user-selected papers from SOSP 2026, and one user-selected SIGCOMM 2026 paper. The SOSP collection includes 11 full-text notes and 7 clearly marked abstract-only notes. The SIGCOMM note uses a full-text arXiv preprint and marks the unverified conference version.

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
