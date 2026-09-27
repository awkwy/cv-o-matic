# CV·O·Matic

**CV Blueprint** — a block-based CV editor with an engineering-blueprint visual
theme (Tornado Cash green, IBM Plex Mono/Sans) for the editor UI, built for
job-hunting. The editor chrome always keeps this look; the printed CV
document itself renders in a separate, switchable CV template — "Blueprint"
(Source Sans 3 / Source Serif 4, the default) or "Classique", a
Jake's-Resume-styled skin with right-aligned dates.

It renders your CV as draggable, resizable blocks laid out on an A4-ratio
page, with:

- A CV template switcher ("Blueprint" / "Classique") that restyles the
  printed document only, leaving the editor UI untouched.
- Drag-and-resize block layout with automatic collision avoidance, so blocks
  never overlap and stay inside the printable page.
- Inline `contenteditable` text editing per block.
- Live ATS-friendliness scoring and a job-description match score (paste a
  job posting to see keyword overlap and missing skills).
- Optional photo upload with client-side downscaling, inline crop/reposition,
  delete, and an enable/disable toggle that reclaims its layout space — no
  upload leaves the browser.
- Draft persistence via `localStorage` — reload the page and your edits are
  still there.
- Desktop-only editing, with a read-only stacked fallback view on narrow
  screens.
- An on-demand "Organiser" action that auto-fits each text block's size and
  font to its content, then re-packs the layout.
- An Overleaf-style collapsible side panel.
- Export to a clean one-page A4 PDF (via the browser's print dialog) and to
  a `.tex` file (downloadable, with copy-to-clipboard as a fallback).

## No backend

This is a **static site** — a single self-contained HTML file with inline
CSS and JavaScript. There is no server, no database, and no build step.
All state lives in the browser via `localStorage`.

## Running it locally

Open the file directly in a browser:

```
open docs/index.html
```

Or serve the `docs/` directory with any static file server, e.g.:

```
npx serve docs
```

## Deployment

This repo is public and serves `docs/` via GitHub Pages from the `main`
branch. Because the repo is public, the deployed page — including any real
name, email, phone number, and CV content typed into it — is reachable by
anyone with the link once Pages is enabled.
