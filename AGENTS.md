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

## Typography: two separate font systems

The tool's own chrome (rulers, buttons, panels, ATS text) uses `--sans`/`--mono`
(IBM Plex Sans/Mono) — that's the deliberate "blueprint" aesthetic, keep it.
The printed CV content (`.cv-block .body`, its read-only mobile mirror, and
`#contact-block`) uses separate `--cv-body`/`--cv-display` variables (Source
Sans 3 / Source Serif 4) so the two never mix. Don't add a third typeface to
the CV content, and don't point CV content at `--mono`.

Each text block can carry a per-block `fontSize` (px) set by the Organiser
action (see below); `h3`/`.meta` sizes inside `.cv-block .body` are in `em`
so they scale with it. Contact and photo blocks don't use `fontSize`.

## Organiser action

The "Organiser" button (rail, Blocs panel) auto-fits each text block's
`rowSpan` to its content and its `fontSize` to fill that box evenly, then
re-packs via the existing `resolveCollisions`/`applyGridPositions`. It's
on-demand only (button click, not live-on-edit) and skips contact/photo
blocks. See `organizeBlocks()` in `docs/index.html` for the tuning constants
(min/max font size, padding, row floor).

Re-packing multiple blocks at once isn't guaranteed collision-free (the
per-block row clamp can still leave a residual overlap), so
`organizeBlocks()` checks every block pair with `rectsOverlap()` afterward
and swaps in a warning toast instead of the success toast when one remains —
keep that check if you touch the function. The button and its hint are
`desktop-only`, matching the other desktop-only editing affordances.

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
