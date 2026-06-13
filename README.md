# nnnsightnnn API portal

Documentation hub for the public JSON data feeds published by the [nnnsightnnn](https://nnnsightnnn.com) tracker apps.

**Live site:** https://nnnsightnnn.github.io/api/

## What this repo is

A single static landing page (`index.html`) that documents the JSON endpoints exposed by each tracker app:

- [liverpool-tracker](https://github.com/Nnnsightnnn/liverpool-tracker) — Liverpool FC (EPL)
- [hawks-tracker](https://github.com/Nnnsightnnn/hawks-tracker) — Atlanta Hawks (NBA)
- [falcons-tracker](https://github.com/Nnnsightnnn/falcons-tracker) — Atlanta Falcons (NFL)
- [braves-tracker](https://github.com/Nnnsightnnn/braves-tracker) — Atlanta Braves (MLB)

Each tracker emits its data as flat JSON at build time under `<tracker-site>/api/v1/<resource>.json`, in addition to rendering the React portal. This repo just describes that contract.

## URL scheme

```
https://nnnsightnnn.github.io/<tracker>/api/v1/<resource>.json
```

`v1` is baked in. Future schema-breaking changes ship as `v2`; existing `v1` paths keep responding for ≥90 days from cutover.

## Editing this page

It's hand-written HTML (`index.html`). No build step, no React, no dependencies. Edit the file, commit, push to `main` — GitHub Pages redeploys in ~30s.

The dark theme matches the trackers' visual language (GitHub-flavored palette: `#0d1117` bg, `#161b22` surface, `#58a6ff` link, `#3fb950`/`#d29922`/`#f85149` for status).

## When a tracker actually ships its v1 endpoints

1. Flip its badge in `index.html` from `badge-planned` ("Rolling out") to `badge-live` ("Live").
2. Append a row to the `Changelog` table at the bottom.
3. If new endpoints landed, add their rows to the tracker's table.

## Stability

Best-effort. One developer. No SLA. Schema breaks get 30 days' notice + a `v2` parallel path. See the page itself for the canonical disclaimer.

## License

MIT.
