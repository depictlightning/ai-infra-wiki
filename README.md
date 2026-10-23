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

## Publishing notes from the research repository

The research repository is the sibling `../awesome-system-papers` on its `ai-infra` branch. Finish and lint a paper note there, then copy only the changed `wiki/` files into the matching paths in this repository. Include affected conference pages, `wiki/index.md`, and `wiki/log.md`; update this README when the collection counts change. Keep PDFs and MinerU output in the research repository.

Review both repositories' working trees before committing. Commit the research changes locally, then commit and push this repository's `ai-infra` branch to `origin`. Its push triggers Cloudflare Pages. Verify the new paper and conference routes with GET requests and check the homepage entry. The research repository's `origin` points to `lqhl/awesome-system-papers`, which has previously rejected pushes from this account; it is not the website remote.
