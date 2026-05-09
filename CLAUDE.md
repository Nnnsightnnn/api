# CLAUDE.md — nnnsightnnn/api

Project-specific guard rails for Claude when working in this repo.

## What this repo is

The documentation hub for the nnnsightnnn tracker apps' public JSON feeds. Single static HTML page deployed to GitHub Pages at https://nnnsightnnn.github.io/api/.

**Not an application.** No React, no Vite, no build step. Hand-written HTML/CSS in `index.html`. That is intentional — the site exists to describe a contract; minimizing its surface area is the design.

## What lives here

- `index.html` — the entire site. Edit, commit, push, done.
- `README.md` — repo-level explainer for developers who land in the GitHub repo.
- `.claude/` — Claude Code scaffolding (memory, hooks, commands).

## What does NOT live here

- The Vite plugin that extracts JSON from each tracker → `nnnsightnnn/tracker-shared`.
- The actual JSON endpoint files → emitted into each tracker's `dist/api/v1/`, served from that tracker's own GitHub Pages site.
- Per-tracker app code → `nnnsightnnn/{liverpool,hawks,falcons,braves}-tracker`.

## Editing rules

- **Stay HTML-only.** Do not introduce a static-site generator, React, Tailwind compile step, or build pipeline. The whole point of this repo is "no build."
- **No external CSS/JS dependencies.** Inline everything. The page must work with no network beyond GitHub Pages itself.
- **Mobile-first.** The trackers' audience reads on phones. Test the breakpoint at 600px width.
- **Match the visual language** of the trackers and `tracker-shared/live-scores.css`: GitHub-flavored dark palette (`#0d1117` / `#161b22` / `#58a6ff` link / `#3fb950` green / `#d29922` amber / `#f85149` red).

## When a tracker ships its v1 endpoints

1. In `index.html`, swap that tracker's badge class from `badge-planned` ("Rolling out") → `badge-live` ("Live").
2. Append a row to the `Changelog` table.
3. Commit direct to `main`. No PR.

## Stability commitment (verbatim — do not soften)

> Best-effort public data feeds. One developer running them as a side project. Schema-breaking changes get a 30-day notice in the changelog and bump to `v2`; existing `v1` paths keep responding for at least 90 days. No uptime SLA. No support contract.

If a future Claude is tempted to upgrade this to a stricter promise, push back.

## Repo conventions (Kenny's defaults)

- Direct-to-main commits. No PRs. No long-lived feature branches.
- Test changes locally (`open index.html` in a browser) before pushing.
- GitHub Pages auto-deploys from `main` via the configured workflow.
