---
inclusion: always
---

# No third-party assets

Rule for any agent working in this repo or driving its MCP server.

## Do

- Build features from what is already here: the workspace crates, the `obscura` CLI/MCP tools, and dependencies already in `Cargo.lock`.
- Draw, lay out, and render with built-in means (obscura-render, CSS, SVG, canvas written in this repo).
- Use web search (the `web-search` MCP server, or `OBSCURA_WEB_SEARCH`) to read and cite, then write original code and text.
- Check a new crate against `deny.toml` before adding it. Ask first; do not add dependencies silently.

## Do not

- Download images, icons, fonts, audio, video, models, datasets, binaries, or code snippets from the web into the repo, `vendor/`, or `assets/`.
- Use `browser_navigate`, `browser_evaluate`, or `browser_pdf` to harvest a site's assets or copy its content wholesale. Reading a page to answer a question is fine.
- Point `--font-dir` at fonts that are not already licensed for this use on this machine.
- Paste licensed or unlicensed third-party code. Re-implement from the spec or docs instead.
- Commit generated screenshots, PDFs, or scraped output (AGENTS.md already forbids generated screenshots and reports).

## When a task seems to need an outside asset

Stop and say so. Offer a built-in alternative (an SVG, CSS, a generated placeholder) or ask the user to supply an asset they own. This repo is Apache-2.0 (see `LICENSE`); everything committed must be compatible with that.
