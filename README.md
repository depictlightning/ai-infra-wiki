# AI Infra Wiki

Quartz site for AI infrastructure research notes, including OSDI, SOSP, MLSys, ICML and other conference papers, selected preprints, technical reports, and curated blog entries. The main reading themes are LLM Inference, LLM Serving, and LLM Training; PD Serving, Inference Simulation, KV Cache Systems, DeepSeek and Speculative Decoding are focused collections. The SOSP 2026 collection includes 8 full-text notes and 6 clearly marked abstract-only notes. The SIGCOMM note has been checked against the conference version supplied by the user. Blog pages label whether their source text has been reviewed.

FAST 2026 and MLSys 2026 are being screened for work directly relevant to LLM training, inference, and serving. Their conference pages distinguish completed reviews from notes still requiring verification; the MLSys list is not complete.

The OSDI 2025 collection currently contains one selected paper, NanoFlow. Its full-text note records unresolved differences between reported figures and plotted results.

The OSDI 2024 collection contains InfiniGen, DistServe, and Sarathi-Serve notes on KV offloading and prefill/decode serving.

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
