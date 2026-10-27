# AI Infra Wiki

Quartz site for 49 AI infrastructure papers selected from OSDI 2026, 18 user-selected papers from SOSP 2026, one user-selected SIGCOMM 2026 paper, two arXiv 2026 preprints (DSpark and HySparse), the Unlimited OCR 2 technical report, curated blog entries, and DeepSeek and vLLM reading themes. The SOSP collection includes 11 full-text notes and 7 clearly marked abstract-only notes. The SIGCOMM note has been checked against the conference version supplied by the user. Blog pages label whether their source text has been reviewed.

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

The research repository is the sibling `../awesome-system-papers` on its `ai-infra` branch. Finish and lint wiki content there, then copy only the changed `wiki/` files into the matching paths in this repository. Include affected theme or conference pages, `wiki/index.md`, and `wiki/log.md`; update this README when the collection scope or counts change. Keep PDFs and MinerU output in the research repository.

Review both repositories' working trees before committing. Commit the research and site changes locally. Before every push, show the user the exact remote, branch, and commits, then obtain explicit approval for that push. Do not reuse approval from earlier work or push while waiting: pushing this repository's `ai-infra` branch triggers a Cloudflare Pages build, and monthly build capacity is limited. After an approved push, confirm that the GitHub remote branch points to the local commit with `git ls-remote`; the user checks Cloudflare Pages, so do not check the build or live routes unless asked. The research repository's `origin` points to `lqhl/awesome-system-papers`, which has previously rejected pushes from this account; it is not the website remote.
