# AGENTS.md

CV Blueprint: a static, single-file CV block-editor. The entire app lives in
`docs/index.html` (HTML + inline CSS + inline JS, no build step, no
dependencies beyond a Google Fonts stylesheet link).

## No backend, on purpose

No server, no database, no build pipeline. All persistence is client-side
`localStorage` (draft state under `cv-blueprint-draft-v1`, rail-collapsed UI
state under `cv-blueprint-rail-collapsed`). Keep it this way unless a future
task explicitly changes that scope.

## PDF export

Uses a `@media print` stylesheet plus `window.print()` — no PDF library.
Verify print output headlessly rather than eyeballing screen CSS:

```
chromium --headless --disable-gpu --no-sandbox --no-pdf-header-footer \
  --print-to-pdf=out.pdf file://<path-to-docs/index.html>
pdftoppm -png -r 150 out.pdf out
```

Then view the resulting PNG. The sibling `cours` project
(`/home/awkwy/firstmate/projects/cours/eleve/render.sh`) uses the same
headless-print-then-rasterize approach.

## LaTeX export

`.tex` export is a real file download (`Blob` + `<a download>`), with a
clipboard-copy button kept as a fallback. Both call the same
`generateLatex()` DOM-walking generator in `docs/index.html` — don't let the
two paths drift into separate generators.

## Deployment

Public repo, GitHub Pages serving `/docs` on `main`. Being public is what
makes free-plan Pages possible — it also means the deployed page (including
whatever real name, email, phone, and CV content is in it) is reachable by
anyone with the link once Pages is enabled.

## Maintaining this file

Keep this file for knowledge useful to almost every future agent session in this project.
Do not repeat what the codebase already shows; point to the authoritative file or command instead.
Prefer rewriting or pruning existing entries over appending new ones.
When updating this file, preserve this bar for all agents and keep entries concise.
