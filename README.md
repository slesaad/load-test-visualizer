# Load Test Visualizer

A single-file, static web dashboard for inspecting load-test JSON reports — drop your file in and see the metrics.

Live: <https://slesaad.github.io/load-test-visualizer/>

## Features

- **Drop a `.json` report** anywhere on the page (or click **⬆ Open file**, or **Try a sample**) to view metrics: virtual users, request rate, latency percentiles, HTTP code mix, per-endpoint breakdown.
- **Drop a matching `.yml`/`.yaml` test config** to overlay configured phases on the trend chart. Each phase paints a tinted band inside the chart and lines up with its label above. If the YAML's duration doesn't match the report, phases are hidden and the indicator warns you.
- **Compare mode** — load two reports side-by-side. A compact, full-width comparison table appears below the panels with every metric, percentage deltas, and a live filter input.
- **Shareable URLs**:
  - `?url=https://…` — fetches the report from any CORS-enabled URL (gist raw, S3, Pages, etc.). Optional `?yaml=…` for a config.
  - `#data=<base64-gzip>` — inline-embeds the report; `&cfg=<base64-gzip>` carries the YAML config alongside.
- **Colorblind-safe palette** (Okabe–Ito), with shape/dash patterns redundant with color.

## Usage

Open `index.html` in a browser. No build, no install.

Files stay in your browser — nothing is uploaded.

## Try it

A sample report (`sample-report.json` + `sample-report.yml`) is included. Once deployed to GitHub Pages, link with:

```
https://<user>.github.io/<repo>/?url=./sample-report.json
```

The matching YAML auto-attaches because it's a sibling file, so the phase strip and Test config populate immediately. There's also a **Try a sample** button in the empty state that does the same thing.

## Deploying

It's a single static file. To host on GitHub Pages:

1. Push `index.html` (plus optional sample files) to a repo.
2. Settings → Pages → Deploy from a branch → `main` / root.
3. Visit `https://<user>.github.io/<repo>/`.

## Browser support

Modern Chromium, Firefox, and Safari. The share-URL fragment encoding uses `CompressionStream` (Chrome 80+, Firefox 113+, Safari 16.4+).
