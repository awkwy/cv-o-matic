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

`.cv-block .body p` (and its read-only mirror) is `text-align: justify` with
`hyphens: auto` — relies on the root `<html lang="fr">` for the hyphenation
dictionary to engage. The `chromium` package in this sandbox has no
hyphenation dictionaries installed, so a headless render here will never show
an actual hyphen break (long unbreakable words just overflow instead) even
though the CSS is correct and end-user browsers with dictionaries installed
hyphenate normally. Don't mistake that sandbox gap for a CSS regression —
verify justify quality by checking word-gap size on real CV content instead
of by looking for hyphens in the screenshot.

`.cv-block .body` also carries `overflow-wrap: break-word` (any text node, not
just `<p>`) so a single word too long for the column breaks instead of
overflowing the box width into the block beside it, and
`hyphenate-limit-chars: 6 3 3` on the `<p>` justify rule keeps hyphenation from
stranding a tiny orphan fragment alone on a line. Print's `.cv-block` /
`.cv-block .body` are `overflow: hidden !important`, not `visible` — a block
whose content doesn't fit its `rowSpan` must clip, not bleed down into the
block below it in the printed page. If you touch either of these, re-verify
with the same reused-profile trick documented in this file's history (inject
oversized text into the saved draft's block `html`, then
`chromium --print-to-pdf` reusing that profile's `--user-data-dir` so the
draft carries over) rather than trusting the CSS by inspection.

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

## CV template switch

The rail's "Modèle de CV" panel (top of the rail) picks between "Blueprint"
(default) and "Classique" (a Jake's-Resume-family skin: off-white page, black
ink, small-caps section titles with a rule beneath, single serif face,
right-aligned dates on Formation/Expérience), stored as `state.template` in
the same `cv-blueprint-draft-v1` draft and normalized to `"blueprint"` on
load if absent/unrecognized (old drafts, first-time visitors). The switch
only repaints the CV document content (canvas page, `.cv-block`, contact
card, print output) via a dedicated `--cv-*` token set defined at `:root`
(Blueprint values) and overridden under `html[data-template="classique"]` —
see the "CV template tokens" comment block in `docs/index.html`'s `<style>`.
These tokens are deliberately separate from the tool chrome's own
`--ink`/`--accent`/`--bg-panel*` (rail, buttons, gauges), so a template
switch never touches the editor UI, only the parts of the DOM that actually
print. `--cv-body`/`--cv-display` (already CV-content-only, see Typography
above) are likewise overridden per template.

Adding a third template means: add its token block under a new
`html[data-template="..."]` selector, add a button to `.template-switch`,
and add its name to the `TEMPLATES` array in `docs/index.html` — no changes
needed to `resolveCollisions`/`organizeBlocks`/export, which are
template-agnostic by construction (they measure/manipulate whatever CSS is
currently active). A template needing a genuinely different layout (e.g. a
fixed sidebar zone, not just new colors/fonts) is a bigger change than this
mechanism supports — see the deferred "Minimal Academic" sidebar family.

Classique also right-aligns a `.meta` line's trailing date into a tabular
column via `splitMetaTabular()`/`applyMetaTabularSplit()` in
`docs/index.html` — a render-time-only DOM split (regex-matched against the
existing text, never rewrites `b.html`) that falls back to plain text when
no date pattern is recognized. It never touches `aggregatedText()`,
`generateLatex()`, or the ATS/match scoring text. The split markup lives on
a separate, never-editable `.body-display` sibling rather than inside the
contenteditable `.body` node itself — see the "Classique's meta-date split"
comment above `.body-display` in `docs/index.html`'s `<style>` for why that
separation, plus the blur-time `contenteditable` reset, are both required.

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

## Photo block: two images, not one

The photo block keeps two data URLs, not one: `originalDataUrl` (a
moderately-higher-res upload, crop source only) and `dataUrl` (the
low-res baked result actually displayed/printed/exported). "Changer"
(upload) sets both. "Rogner" opens an inline pan/zoom crop dialog
(`openCropOverlay()` in `docs/index.html`) sized to the block's own
on-screen aspect ratio; "Valider" re-bakes `dataUrl` from
`originalDataUrl` at the current pan/zoom and leaves `originalDataUrl`
untouched, so cropping is repeatable. The delete action is a small red
`.photo-delete-btn` (×) pinned top-right of the thumbnail — deliberately
separate from the bottom `.photo-toolbar` (Changer/Rogner), not a third
toolbar button — and nulls both URLs back to the empty-placeholder state.
A saved draft from before this existed has `dataUrl` but no
`originalDataUrl` — the toolbar hides "Rogner" rather than crash; keep
that fallback if you touch this block's rendering.

The photo block also carries an `enabled` boolean (default `true`, toggled
from a checkbox next to "Photo" in the rail's Blocs panel via
`setPhotoEnabled()` in `docs/index.html`). Disabling it drops the block from
layout entirely — it's excluded from `layoutBlocks()`, the single filter
every layout-facing path (`renderCanvas`, `resolveCollisions`) shares — while
keeping `dataUrl`/`originalDataUrl` in state so re-enabling restores the
photo without a re-upload. Disabling auto-expands Contact from its default
`colSpan:9` to `colSpan:12` (only if Contact is still at that untouched
default — a manually resized Contact is left alone); re-enabling reverses
that and resets the photo block to its default grid slot, then re-runs
`resolveCollisions` rather than hand-placing it. `aggregatedText()`, the ATS
"photo intégrée" check, the readonly/mobile mirror, and both export
generators all gate on `photoBlock.enabled` (not just `dataUrl`) so a
disabled photo is excluded everywhere consistently. A saved draft from
before this field existed has no `enabled` — normalized to `true` on load.

## LaTeX export

`.tex` export is a real file download (`Blob` + `<a download>`), with a
clipboard-copy button kept as a fallback. Both call the same
`generateLatex()` DOM-walking generator in `docs/index.html` — don't let the
two paths drift into separate generators.

## Sidebar layout: `.rail` leaving the grid

`.layout` is a 3-column CSS Grid (`.rail` / `.splitter` / `.canvas-wrap`).
`html.rail-collapsed .rail { display: none }` removes `.rail` from grid
auto-placement entirely, so `.splitter`/`.canvas-wrap` need an explicit
`grid-column` (see the `@media (min-width: 880px)` rule right after
`html.rail-collapsed .layout`) or they silently shift into the wrong tracks
when the rail collapses. If you touch `.layout`'s columns or the
rail-collapse toggle, re-check that rule and its `@media print` override,
and verify by actually clicking `#rail-toggle` in a live/headless browser —
this class of bug doesn't show up in a static screenshot of either steady
state.

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
