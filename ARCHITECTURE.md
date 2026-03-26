# Architecture

## Overview

Newsletter Archive is a static site generator + client-side app for browsing ~14,000 email newsletters. It follows a "build once, render client-side" pattern with zero runtime dependencies.

## Data Flow

```
Source repo (../newsletters/)     Build script              GitHub Pages
  newsletters/                     scripts/build_site.py      Served as static files
    {name}/                          |
      {hash}/                        +-> emails/{name}/{hash}/{hash}.html  (copied HTML)
        {hash}.html                  +-> data/index.json                   (manifest)
        {hash}.md (metadata)         +-> index.html, newsletter.html, ...  (templates)
        {hash}.txt
```

1. Build script walks `../newsletters/`, parses YAML front matter from `.md` files
2. Copies only `.html` files to `emails/` (no `.md` or `.txt` — keeps size manageable)
3. Generates `data/index.json` with all email metadata (subject, from, date, file path)
4. Copies template files to repo root

## Key Design Decisions

- **No static site generator (Jekyll/Hugo)**: 14K HTML files already render perfectly. We only need ~70 navigation pages + 1 JSON manifest. Python stdlib handles this in ~10 seconds.
- **Client-side rendering**: `index.json` manifest is loaded once, then all navigation/search is client-side. No server needed.
- **Iframe email viewer**: Original HTML emails are loaded in sandboxed iframes to prevent CSS conflicts and preserve original formatting.
- **Repo root deployment**: Built files go directly to repo root (not `dist/`). The entire repo is deployed as a static site via GitHub Pages.
- **Git LFS for emails**: The 13,652 HTML email files (~883 MB) are stored via Git LFS to keep clone size small (~4 MB pack vs 137 MB without LFS). `data/index.json` (3.1 MB) stays in regular git for delta compression and native diffing. The CI workflow caches `.git/lfs/` to minimize bandwidth usage on GitHub's free tier (1 GB/month).

## File Structure

```
scripts/build_site.py       # Build script (reads source, generates output)
templates/                   # Source templates (copied to root on build)
  index.html                 # Homepage template
  newsletter.html            # Newsletter listing template
  view.html                  # Email viewer template
  bookmarks.html             # Bookmarks listing page
  style.css                  # Shared styles
  app.js                     # Client-side search/nav/storage
data/index.json              # Generated manifest (all email metadata)
emails/                      # Copied HTML email files
.github/workflows/deploy.yml # GitHub Pages deployment
```

## Client-Side Architecture

`app.js` is organized as an IIFE module (`App`) with methods:
- `initHomepage()` — loads manifest, renders newsletter card grid with read counts, binds search
- `initNewsletter()` — filters manifest by newsletter name, renders date-sorted email list with read/bookmark state
- `initViewer()` — sets iframe src, auto-marks email as read, loads prev/next navigation, bookmark toggle
- `initBookmarks()` — loads bookmarked emails from localStorage, renders with newsletter labels and search
- `initThemeToggle()` — initializes the theme toggle button (home page only), cycles through system/light/dark
- `initKeyboard()` — `/` to focus search, `Escape` to blur

Internal `Store` module provides localStorage-backed persistence with in-memory cache:
- **Set-based** (`_get`/`_save`): `nl_read` (read emails), `nl_bookmarks` (bookmarked emails) — JSON arrays serialized to Sets for O(1) lookups
- **Map-based** (`_getMap`/`_saveMap`): `nl_card_colors` (newsletter name → hex color) — JSON objects for key-value storage

All data comes from a single `data/index.json` fetch cached in memory.

### Theme Management

Three-state theme toggle (System / Light / Dark) on the home page header. Preference persisted in `localStorage` key `nl_theme` (`"light"`, `"dark"`, or absent for system auto). CSS uses `html[data-theme="dark"]` and `html[data-theme="light"]` attribute selectors to override the default `@media (prefers-color-scheme: dark)` media query when the user makes a manual choice. A synchronous inline `<script>` in every page's `<head>` sets the `data-theme` attribute before the stylesheet loads to prevent flash of wrong theme (FOUC). Internal `Theme` module in `app.js` manages state and cycling. Toggle button only on home page; preference respected on all pages.

### Card Color Picker

Homepage cards have a color picker swatch (top-right corner) that lets users customize each card's background. Uses a `<label>` wrapping a hidden `<input type="color">` — the label is the styled 16px circle, the native input provides the OS color picker. Card background is set via CSS custom property `--card-bg` (not inline `backgroundColor`) so that hover states in the stylesheet can reference and darken/lighten the color with `color-mix()`. Colors persist in `nl_card_colors` localStorage via the `Store._getMap`/`_saveMap` key-value methods. Event delegation on the grid container (`bindCardColorPickers`) prevents `<a>` navigation on click and live-updates the card on `input` events.

### List-Level Actions (Event Delegation)

Email list rows have inline action buttons (read toggle, bookmark toggle) that modify state without navigating to the viewer. This uses **event delegation**: a single click listener is attached to the list container (not per-button), and `e.target.closest("[data-action]")` identifies the clicked button. `e.preventDefault()` + `e.stopPropagation()` prevents the parent `<a>` link from navigating. The listener is bound once per container via a `_actionsListenerBound` flag to survive re-renders.

Row structure:
```html
<a class="email-item" data-file="emails/...">
  <span class="email-item__content">   <!-- baseline-aligned text wrapper -->
    <span class="email-item__date">...</span>
    <span class="email-item__subject">...</span>
  </span>
  <span class="email-item__actions">   <!-- right-aligned toggle buttons -->
    <button data-action="toggle-read">...</button>
    <button data-action="toggle-bookmark">...</button>
  </span>
</a>
```
