# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

けいすけの47都道府県たび ("Keisuke's 47 Prefectures Trip") — a Japanese-language PWA for tracking a family's progress visiting all 47 prefectures of Japan with their son Keisuke (born 2024-10-02). All UI copy is in Japanese (hiragana/kanji); preserve the existing casual tone (e.g. 関西弁 "〜してね", "〜やで") when editing user-facing strings.

There is no build system, package manager, or test suite. This is a pure static site: HTML + inline `<style>` + inline `<script>`.

## Architecture

### Files
- `index.html` (~4000 lines) — the entire app. Contains all CSS, markup, and JS inline in one file.
- `roadmap.html` — standalone decorative 10-year visit roadmap, linked from the plan view.
- `sw.js` — service worker. Network-first for HTML (to avoid stale app versions), cache-first for other assets. **Bump the `CACHE` constant (`keisuke47-v1` → `v2` …) whenever cached asset names/contents need a forced refresh.**
- `manifest.json` — PWA manifest. Must stay in sync with `sw.js` ASSETS list and the icon files.
- `.github/workflows/deploy.yml` — deploys the repo root to GitHub Pages on push to `main`. No build step.

### Single-file app structure (`index.html`)
The script at the bottom is organized in labeled sections (`// ===== Data =====`, `// ===== State =====`, `// ===== Plan View =====`, `// ===== Shiori =====`, `// ===== Firebase Cloud Sync =====`, etc.). When adding features, follow this sectioning.

Four top-level views switched by `switchNav(view)` — `home` (map/list tabs), `plan`, `shiori`, `stats`. `switchNav` manually toggles `.active` on DOM nodes and shows/hides header/progress sections; there is no framework/router.

### Data model & persistence
All state lives in module-level `let` globals, persisted to `localStorage` as JSON:

| Global       | localStorage key            | Shape |
|--------------|-----------------------------|-------|
| `records`    | `keisuke47_records`         | `{ [prefName]: { visited, date, memo, photos[], claudeInfo, claudeRaw, ... } }` |
| `tripPlans`  | `keisuke47_tripplans`       | `{ [year]: [ { pref, month, spots:[3], shioriId? } ×5 ] }` |
| `shioriData` | `keisuke47_shiori`          | `[ { id, title, pref, startDate, endDate, days:[{date, items:[{time,place,url,memo,photos?}]}], packing?, budget?, weather? } ]` |
| —            | `keisuke47_cloudkey`        | saved Firestore sync passphrase (alnum + `_-`, ≥6 chars) |

Every mutation must call the matching `save*()` function (`saveData`, `saveTripPlans`, `saveShiori`) to flush to `localStorage`. After bulk changes call `updateAll()` to re-render progress, map, list, and backup stats.

### Prefecture map rendering
`REGIONS` groups the 47 prefectures into 8 regions with colors. `GRID_POS` maps each prefecture to `[row, col]` on an 11-col × 15-row CSS grid. Overlaps are intentionally fixed up by later reassignments at the bottom of the `GRID_POS` block (e.g. 滋賀/岐阜/京都/大阪) — if you reorganize, preserve the final positions or the map will collide.

### Cloud sync (Firebase)
`FIREBASE_CONFIG` is embedded in `index.html` (public web API key — acceptable per Firebase's model, security is enforced by Firestore rules). Data is stored at `users/<cloudkey>` as a single JSON blob. The Firebase SDK is lazy-loaded via ES module `import()` from `gstatic.com`; `initFirebase()` memoizes the DB handle. The "cloudkey" is the user-chosen passphrase, not an auth credential — two users with the same key share data.

### Photos
Photos are resized client-side (canvas, `MAX=400`px) to base64 data URLs before being stored in `localStorage`, which is the main space constraint. Keep this resize step for any new photo-upload path or users will hit the ~5MB quota quickly.

### Claude integration (user-facing, not API)
`getClaudePrompt(pref)` produces a Japanese prompt template the user copies into Claude.ai. `parseClaudeResponse(text)` extracts sections delimited by `【...】` headers. When editing either, they must stay structurally compatible — the parser regex matches the exact section headers emitted by the prompt.

## Development

### Run locally
No build. Serve the directory over HTTP (service worker and ES module imports require a real origin, not `file://`):

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

### Deploying
Push to `main` → GitHub Actions auto-deploys to GitHub Pages via `.github/workflows/deploy.yml`. No staging environment.

### Cache busting
If users report stale UI after a deploy, the service worker's network-first HTML strategy normally handles it, but for asset changes (icons, manifest) bump `CACHE` in `sw.js`.

## Conventions

- **Japanese-first UI.** All user-visible strings are Japanese. New strings should match surrounding tone (casual, child-friendly, some Kansai-ben).
- **No dependencies.** Don't introduce npm, bundlers, frameworks, or CSS preprocessors — everything stays inline in `index.html`. Firebase is the only runtime dep and it's loaded by dynamic `import()` on demand.
- **Inline `onclick=` handlers** are the established event-binding style. Keep using them for consistency rather than mixing in `addEventListener`.
- **Commit messages** are written in Japanese (see `git log`). Match that style when committing.
- **The birthday constant** `BIRTHDAY = new Date(2024, 9, 2)` drives age calculations throughout the app; don't hardcode ages.
