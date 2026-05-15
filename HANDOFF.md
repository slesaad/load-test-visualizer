# Handoff — Load Test Visualizer

This document orients a future agent (or human) picking up this project. It assumes you've read `README.md` for the user-facing summary.

## 1. What this is

A **single static HTML file** (`index.html`) that renders a dashboard from an Artillery-style load-test JSON report (output of `artillery run --output report.json`). Hosted statically (GitHub Pages). Users drop their JSON in; no backend.

The original development happened in a different repo at `~/repo/IMPACT/fm-inference-service/stress_test/` against a real Artillery test suite; this repo (`load-test-visualizer`) is the cleaned, public-deployable distribution: just `index.html` + `README.md`.

The word "Artillery" is intentionally absent from user-visible strings — see §8 (Trademark cleanup) below.

## 2. File layout

```
load-test-visualizer/
  index.html           ← everything: HTML + inline CSS + inline JS
  sample-report.json   ← bundled demo report (Try-a-sample button + ?url=./…)
  sample-report.yml    ← matching config (auto-attaches as sibling)
  README.md            ← user-facing overview
  HANDOFF.md           ← this file (not yet committed by default)
```

`sample-report.json` is a small real test result (~47 KB, single phase, public API endpoint names). `sample-report.yml` is a one-phase config that maps to it. They're loaded by the "Try a sample" button in the empty state (which calls `loadFromUrl('./sample-report.json', panel)`). The YAML auto-probe finds the sibling `.yml` automatically.

No build step, no package manager, no bundler. External runtime deps (CDN):

- Chart.js (line + doughnut charts)
- js-yaml (parsing dropped YAML configs)
- Google Fonts (Manrope, JetBrains Mono, Fraunces)

If you ever want to vendor these, save copies next to `index.html` and rewrite the `<script>`/`<link>` tags.

## 3. Data model — what's in an Artillery JSON

Two top-level keys: `aggregate` and `intermediate[]`. Phase config is **not** in the JSON; that's why we support optional YAML companion files.

```
{
  "aggregate": {
    "counters":   { "http.requests": N, "http.codes.200": N,
                    "plugins.metrics-by-endpoint.<URL>.codes.<code>": N,
                    "vusers.created": N, "vusers.completed": N, "vusers.failed": N, ... },
    "rates":      { "http.request_rate": N },
    "summaries":  { "http.response_time": { min, mean, p50, p75, p90, p95, p99, max, count } },
    "firstCounterAt": ms-since-epoch,
    "lastCounterAt":  ms-since-epoch
  },
  "intermediate": [
    { same shape as aggregate, but for a ~10-second window },
    ...
  ]
}
```

Per-endpoint metrics use keys like `plugins.metrics-by-endpoint.GET /inferences/{id}.codes.200`.

## 4. Architecture inside `index.html`

The single file is organized top-to-bottom as:

1. `<head>` — title, CDN script/font links, big `<style>` block.
2. `<body>`
   - **Topbar** — Open / Share buttons, Compare toggle.
   - **`#drop-overlay`** — floating banner shown during drag.
   - **`.layout`** — flex grid: one or two `.main` panels + a `.side` (Test config).
   - **`<template id="main-template">`** — DOM that's cloned per panel (A, optionally B).
3. `<script>` — all the logic, executed at end of body. Entry point is `init()`.

### Panel lifecycle

Each panel is an object:

```js
{ root, select, closeBtn, tag,
  currentReport, sourceUrl,            // identity
  lastData,                            // most recent parsed JSON
  yaml, yamlPath, yamlSource,          // attached config (auto | manual)
  // root.__charts holds the Chart.js instances (destroyed/replaced on re-render)
}
```

`render(data, root)` is the big function that populates a panel's DOM from the parsed JSON. It uses scoped `root.querySelector('.js-*')` lookups, so multiple panels coexist without ID collisions.

### Key classes/IDs (do not rename without updating JS)

- `js-` prefixed classes inside the template: `js-report-select`, `js-panel-tag`, `js-close-btn`, `js-vusers-total`, `js-vusers-completed`, `js-vusers-completed-pct`, `js-vusers-failed`, `js-vusers-failed-pct`, `js-avg-rps`, `js-peak-rps`, `js-phase-timeline`, `js-trend-chart`, `js-perc-list`, `js-req-count`, `js-resp-count`, `js-codes-chart`, `js-codes-list`, `js-url-list`, `js-load-more-btn`, `js-url-foot-label`, `js-yaml-indicator`, `js-empty-state`, `js-empty-open`, `js-empty-sample`, `js-test-config`.
- IDs: `tab-compare`, `drop-overlay`, `layout`, `main-template`, `file-input`, `open-btn`, `share-btn`, `share-popover`, `compare-table`.
- Structural classes: `main`, `main.secondary`, `main.has-data`, `main.drop-target`, `layout`, `layout.compare`, `donut-box`, `badge.bad`, `compare-table-panel`, `metric-row`.

### Data classes rendered into `innerHTML` (stylable but the JS expects them)

- Percentile rows: `.perc-row`, `.perc-track`, `.perc-bar`, `.pname`, `.pval`
- HTTP code rows: `.crow`, `.cdot`
- URL rows: `.url-row`, `.ep`, `.pill`
- Status badges: `.badge`, `.badge.bad`, `.bdot`
- Phase segments: `.phase-seg.up`, `.phase-seg.down`, `.phase-seg.flat`, `.phase-empty`
- Test config code spans: `.k`, `.v`

## 5. Features and where to find them

| Feature | Function(s) |
|---|---|
| File picker + drag-drop ingestion | `ingestFile`, drop handler in `init()` |
| "Try a sample" button (empty state) | wired in `createPanel()` → `loadFromUrl('./sample-report.json', panel)` |
| Load report from a URL | `loadFromUrl` |
| Switch between session-loaded reports | `switchToReport`, `addSessionReport`, `syncSelectFromSession` |
| Compare mode (two panels side-by-side) | `enterCompare`, `exitCompare`, `createPanel`, `removePanel` |
| **Comparison table** (full-width, below the panels in compare mode) | `renderComparison`, `summarize`, `row`, helpers `fmtNum`/`fmtBytes`/`fmtDurationSec`/`deltaText`; called from `renderSafe` and `enterCompare` |
| Trend chart (4 lines + dash/dot variants for colorblind redundancy) | inside `render()` — search "Trend chart" |
| HTTP percentile bars | inside `render()` — search "HTTP performance: percentiles" |
| HTTP codes donut | inside `render()` — search "HTTP codes donut" |
| Requests-by-URL breakdown with load-more | inside `render()` — search "Requests breakdown by URL" |
| YAML config auto-probe | `probeYaml`, `fetchYaml` (returns `{text, doc}`), `yamlPhasesFor` |
| YAML drag-drop attachment (manual) | drop handler routes by extension to `ingestFile` → YAML branch; stores raw text as `panel.yamlText` so the Share button can re-emit it |
| Phase timeline strip | `renderPhaseTimeline`, called from `render()` |
| Phase background bands inside the trend chart, padding alignment of the strip with `chart.chartArea` | `phaseBgPlugin` (Chart.js plugin registered globally near the top of the script) |
| Phase-config / report duration mismatch detection (hide phases if YAML duration differs >10% from report) | `render()` (computes `phaseMismatch`, sets `root.__phaseMismatch`), `renderSafe` (copies to `panel.yamlMismatch`), `updateYamlIndicator` (renders the warning state) |
| Sidebar Test config | `renderTestConfig`, called from `render()` |
| Share URL generation (JSON + YAML) | `openSharePopover`, `gzipB64Url`, `ungzipFromB64Url` |
| Inbound share URL handling | `init()` reads `?url=`, `?yaml=`, hash params `data=` and `cfg=` |

## 6. Design decisions and constraints

- **Single file, no build.** Adding a bundler/framework requires a build step and breaks the "drop on GitHub Pages and go" promise. Keep that property unless there's strong reason.
- **Files stay local.** Drag-dropped data is parsed in-browser and never sent anywhere. Inline share URLs use `#data=` (fragment isn't sent to servers). `?url=` fetches from a third-party CORS-enabled host of the user's choosing.
- **Colorblind-safe palette.** Okabe-Ito. Don't reintroduce red/green pairings for status — failure badges use orange `#D55E00` plus a `!` glyph. Trend chart lines are distinguished by **color + dash pattern + point shape** (three redundant channels). If you add more chart series, follow the same approach.
- **Chart.js sizing gotcha.** The donut canvas is wrapped in a fixed-size `.donut-box` because Chart.js measures the canvas's *parent* — using a flex/grid sibling cell stretches the canvas non-square. Don't remove that wrapper.
- **Heuristic phase inference was removed.** Phases come only from an attached YAML config (auto-probed sibling or manual drop). If you want to bring inference back, keep it behind a clear UI toggle so users know it's a guess.
- **Manual phase form was removed.** Tried two iterations (drag-to-select; form-based table). User found both clumsy. If revisiting, prefer minimal UI and don't store in `localStorage` unless asked.
- **Share URL fragment encoding.** Gzip + URL-safe base64 via `CompressionStream`. Browser support is recent (Chrome 80, FF 113, Safari 16.4). Warn at ~32 KB; nothing actively prevents going larger but Chrome's hard URL limit is ~2 MB.
- **Share URLs carry the YAML config too.** When a YAML is attached, its raw text is gzip-encoded into `&cfg=` (in the fragment, alongside `data=`) or referenced via `?yaml=<url>` for URL-mode shares. `init()` parses both. Inbound `cfg=` takes precedence over `yaml=`.
- **Phases align with chart x-axis via Chart.js plugin.** `phaseBgPlugin` (`beforeDatasetsDraw` + `afterRender`) paints colored phase bands inside `chart.chartArea` and sets the phase-timeline strip's `padding-left`/`padding-right` to match `chart.chartArea.left` / `canvasWidth - chart.chartArea.right` so the pills above the chart line up with their bands. Re-runs automatically on resize. Don't put inline `padding-*` on `.phase-timeline` in CSS — the plugin will overwrite it on render.
- **Phase mismatch suppression.** If the attached YAML's total phase duration differs from the report's observed duration by more than 10% (`PHASE_TOLERANCE` in `render()`), phases are hidden (no pills, no chart bands) and the config indicator flips to a warning state. Don't silently re-enable — surface the mismatch so users don't trust a wrong overlay.
- **Auto-discovery of disk files is gone.** This was for local Python `http.server` dev; broke on GitHub Pages anyway (no directory listing). Don't add it back; use Share URLs or the Open/Drop flow.

## 7. Local dev and testing

```bash
cd ~/repo/load-test-visualizer
python3 -m http.server 8765   # or any static server
open http://localhost:8765/
```

There's no test suite. Smoke tests when you change something:

1. Empty state shows "Drop your load test report" with **Choose a file** and **Try a sample** buttons. Click Try a sample → `sample-report.json` loads and its sibling `sample-report.yml` auto-attaches (phase timeline appears, indicator goes green).
2. Drop a JSON in — dashboard renders. Check load summary, trend chart, percentile bars, donut, URL breakdown.
3. Drop a YAML config — phase timeline appears, config indicator goes green. Each phase paints a tinted band inside the chart plot area; the pill above each band lines up horizontally with it (alignment driven by `phaseBgPlugin` reading `chart.chartArea`).
4. **Mismatch case**: drop a YAML whose total phase duration differs from the report by more than 10% — phases and bands disappear, the config indicator turns orange with a `⚠` and an `i` tooltip explaining the mismatch.
5. Click Compare — second panel opens with its own selector + drop target. A full-width comparison table appears below both panels with metric / A / B columns and percentage deltas (green better, orange worse). Filter input narrows rows live.
6. Click Share with no source URL — link includes `#data=…` (and `&cfg=…` if a YAML is attached). Paste in a new tab and the report + config both reload.
7. Open `?url=…` to a CORS-enabled JSON — loads automatically. `?yaml=…` likewise attaches a YAML by URL.
8. Try a large report (> 30 K intermediates if you have one) and confirm Share warns about size.

JS syntax check helper:

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>\\n([\\s\\S]*?)<\\/script>/);new Function(m[1]);console.log('OK')"
```

## 9. User preferences (carry-over notes)

- **Design preference**: dark, distinctive, colorblind-safe. Avoid generic dashboard aesthetics. The current Okabe-Ito palette + Manrope/JetBrains Mono/Fraunces type stack is the agreed baseline.

## 10. Likely next steps

- **Share-URL polish**: shorter encoding (e.g., msgpack + brotli) for tighter URLs; explicit "upload to gist" helper using the GitHub gist API to bypass the URL-size limit.
- **Comparison-table polish**: sortable columns; collapsible per-endpoint sections (rows can be plentiful); pin top-N "worst regressions" at the top; CSV export.
- **Multi-report compare > 2**: currently the layout supports A and B only. The code paths assume at most two panels in places (`panels[1]`, `enterCompare` checks `>= 2`). The compare-table would also need column generalization.
- **Different runners**: support k6 / Vegeta / Locust / Gatling JSON shapes. They share the basic concepts (latency percentiles, request counts, per-endpoint breakdown) but the key names differ. A small adapter at the top of `render()` and `summarize()` would normalize.
- **Export**: button to save the current view as PNG/SVG; or export the parsed metrics summary as CSV.
- **Settings persistence**: the user has not asked for any persistence; current behavior (lose everything on reload) is intentional simplicity.

## 11. Things that have been tried and abandoned

So you don't relitigate:

- **Inline directory listing** to auto-populate a file picker — only worked with Python's http.server, broke on GH Pages. Removed.
- **Heuristic phase inference from arrival-rate signal** — worked, but produced misleading "guesses" that users mistook for ground truth. Removed in favor of YAML-only phases.
- **Drag-to-select phase definition on the chart** — implemented and removed; felt clumsy.
- **Form-based phase editor (table with row per phase)** — implemented and removed for the same reason.
- **`localStorage` persistence of manual phases** — removed alongside the form editor.

If you bring back any of these, do it behind a clear feature flag/toggle.
