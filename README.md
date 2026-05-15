# Load Test Visualizer

A single-file, static web dashboard for inspecting load-test JSON reports — drop your file in and see the metrics.

## Features

- Drop a `.json` report anywhere on the page (or click **Open file**) to view metrics: virtual users, request rate, latency percentiles, HTTP code mix, per-endpoint breakdown.
- Drop a matching `.yml`/`.yaml` test config to overlay the configured phases on the trend chart.
- Side-by-side compare mode for two reports.
- Shareable URLs:
  - `?url=https://…` — fetches the report from any CORS-enabled URL (gist raw, S3, Pages, etc.).
  - `#data=<base64-gzip>` — inline-embeds small reports.
- Colorblind-safe palette (Okabe–Ito), with shape/dash patterns redundant with color.

## Usage

Open `index.html` in a browser. No build, no install.

Files stay in your browser — nothing is uploaded.

## Try it

A sample report (`sample-report.json` + `sample-report.yml`) is included. Once deployed to GitHub Pages, link with:

```
https://<user>.github.io/<repo>/?url=./sample-report.json
```

The matching YAML auto-attaches because it's a sibling file, so the phase strip and Test config populate immediately.

## Deploying

It's a single static file. To host on GitHub Pages:

1. Push `index.html` to a repo.
2. Settings → Pages → Deploy from a branch → `main` / root.
3. Visit `https://<user>.github.io/<repo>/`.

## Browser support

Modern Chromium, Firefox, and Safari. The share-URL fragment encoding uses `CompressionStream` (Chrome 80+, Firefox 113+, Safari 16.4+).
