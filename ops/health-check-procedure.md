---
name: api-portal-health-check
description: Daily health check for the nnnsightnnn tracker API portal — verifies each tracker's /api/v1/index.json, runs field-level contracts against every documented endpoint to catch 200-but-hollow payloads and stale manifests, refreshes the 'Last verified' footer, flips badges only on genuine outages, and iMessages on new regressions only.
---

Daily health check for the nnnsightnnn tracker API portal at https://nnnsightnnn.github.io/api/. This runs after all four tracker-update tasks (liverpool 04:02, hawks 04:14, falcons 04:30, braves 04:40) so deploys have settled by now.

The portal lives at `~/api/` on Kenny's MacBook (repo: `nnnsightnnn/api`, GitHub Pages auto-deploys from `main`). The doc hub itself is static HTML — this task's job is to keep its accuracy/health signal honest.

## What this task does

For each tracker in [liverpool-tracker, hawks-tracker, braves-tracker, falcons-tracker], fetch `https://nnnsightnnn.github.io/{tracker}/api/v1/index.json` and capture:
- HTTP status code
- The manifest's `commit` field
- The count and resource names of the manifest's `endpoints` array
- Whether the response was JSON-parseable
- The manifest's `generatedAt`, converted to an age in hours

Then fetch each tracker's individual resources and run the **field contracts** in the script below. A 200 response carrying an empty or null payload is the failure mode this task exists to catch: the endpoint is up, the resource names are unchanged, and the data is hollow. `falcons-tracker/next-game.json` served `null` for months while every check reported green, because nothing looked inside.

If all four return 200 + parseable JSON, pass their field contracts, and are fresh: update only the "Last verified" footer line in `~/api/index.html` and commit. Do not touch badges or alert.

If any tracker is not 200 or returns malformed JSON: flip its badge in `~/api/index.html` from `badge-live`/Live to `badge-deprecated`/Down, append a row to the changelog table noting the regression with today's date, and send Kenny an iMessage with the failure details.

If a tracker recovers (was deprecated, now returns 200): flip its badge back to `badge-live`/Live and append a recovery row to the changelog. Do not iMessage — recovery is silent.

If the manifest's endpoint count or resource names have drifted from the documented set in `index.html`: append an "endpoint drift" entry to the changelog, but do NOT change the badge. Also iMessage Kenny so he can decide whether the docs need updating.

If a tracker is **hollow** (serving 200 but failing a field contract) or **stale** (manifest `generatedAt` older than 36 hours, meaning its own update task probably didn't run): append a changelog row, but do NOT flip the badge — the endpoint is genuinely up, so calling it Down would be a lie. iMessage only on the *transition* into that state, judged against the previous run's `.health/last.json`. A condition that is still true tomorrow is not news, and a daily nag trains Kenny to ignore the alert.

## Step-by-step

### Step 1: Confirm the repo exists

```bash
test -d ~/api/.git && echo OK || { echo "MISSING ~/api repo"; exit 1; }
```

If MISSING, stop and report — don't try to recreate.

### Step 2: Run the checks

Write the script under `~/api/.health/` rather than `/tmp` — the file tooling available to this task is scoped to `/Users/kenny`, and a `/tmp` path will be refused.

```bash
mkdir -p ~/api/.health
cd ~/api

cat > ~/api/.health/check.js <<'JSEOF'
const TRACKERS = ['liverpool-tracker', 'hawks-tracker', 'braves-tracker', 'falcons-tracker'];
const DOCUMENTED_ENDPOINTS = {
  'liverpool-tracker': ['squad', 'news-digest', 'results', 'next-match', 'standings', 'standings-commentary', 'lineup'],
  'hawks-tracker':     ['roster', 'news-digest', 'results', 'next-game', 'playoff-series', 'standings'],
  'braves-tracker':    ['roster', 'news-digest', 'results', 'next-game', 'upcoming-schedule', 'standings'],
  'falcons-tracker':   ['roster', 'news-digest', 'next-game', 'draft', 'calendar'],
};

// Resources that are legitimately null when nothing is happening.
// Do NOT add falcons/next-game here — that one being null IS the bug.
const ALLOW_NULL = {
  'hawks-tracker': ['playoff-series'],
};

const STALE_HOURS = 36;

// Generic contract: a 200 that carries nothing is a failure.
function generic(v) {
  if (v === null || v === undefined) return 'null payload';
  if (Array.isArray(v) && v.length === 0) return 'empty array';
  if (typeof v === 'object' && !Array.isArray(v) && Object.keys(v).length === 0) return 'empty object';
  return null;
}

const filled = (o, k) => o && typeof o === 'object' && o[k] !== undefined && o[k] !== null && o[k] !== '';
const listOf = (v, n) => Array.isArray(v) && v.length >= n;

// Specific contracts, checked only when the generic one passes.
// Keep these loose enough to survive normal editorial variation and tight
// enough to catch a renamed field or a collapsed payload.
const CONTRACTS = {
  'liverpool-tracker': {
    'squad':       v => listOf(v, 15) && filled(v[0], 'name') && filled(v[0], 'position') ? null : 'expected >=15 players with name+position',
    'news-digest': v => typeof v.summary === 'string' && v.summary.length > 200 && listOf(v.keyTopics, 1) ? null : 'missing summary (>200 chars) or keyTopics',
    'results':     v => listOf(v, 1) && filled(v[0], 'opponent') && filled(v[0], 'score') ? null : 'missing opponent/score on first row',
    'next-match':  v => filled(v, 'opponent') && filled(v, 'date') ? null : 'missing opponent/date',
    'standings':   v => listOf(v, 4) && v[0].pts !== undefined && filled(v[0], 'team') ? null : 'missing pts/team',
  },
  'hawks-tracker': {
    'roster':      v => listOf(v, 10) && filled(v[0], 'name') && filled(v[0], 'position') ? null : 'expected >=10 players with name+position',
    'news-digest': v => typeof v.summary === 'string' && v.summary.length > 200 && listOf(v.keyTopics, 1) ? null : 'missing summary (>200 chars) or keyTopics',
    'results':     v => listOf(v, 1) && filled(v[0], 'opponent') && filled(v[0], 'score') ? null : 'missing opponent/score on first row',
    'next-game':   v => filled(v, 'opponent') && filled(v, 'date') ? null : 'missing opponent/date',
    'standings':   v => listOf(v, 4) && filled(v[0], 'record') && filled(v[0], 'team') ? null : 'missing record/team',
  },
  'braves-tracker': {
    'roster':      v => listOf(v, 20) && filled(v[0], 'name') && filled(v[0], 'position') ? null : 'expected >=20 players with name+position',
    'news-digest': v => typeof v.summary === 'string' && v.summary.length > 200 && listOf(v.keyTopics, 1) ? null : 'missing summary (>200 chars) or keyTopics',
    'results':     v => listOf(v, 1) && filled(v[0], 'opp') && v[0].atlScore !== undefined ? null : 'missing opp/atlScore on first row',
    'next-game':   v => filled(v, 'opp') && filled(v, 'date') ? null : 'missing opp/date',
    'standings':   v => listOf(v, 4) && v[0].w !== undefined && filled(v[0], 'team') ? null : 'missing w/team',
  },
  'falcons-tracker': {
    'roster':      v => listOf(v, 40) && filled(v[0], 'name') && filled(v[0], 'position') ? null : 'expected >=40 players with name+position',
    'news-digest': v => v.cover && filled(v.cover, 'deck') && listOf(v.topics, 1) ? null : 'missing cover.deck or topics',
    'next-game':   v => filled(v, 'opponent') && filled(v, 'date') ? null : 'missing opponent/date',
    'draft':       v => filled(v, 'draftYear') && listOf(v.falconsPicks, 1) ? null : 'missing draftYear/falconsPicks',
    'calendar':    v => listOf(v, 1) && filled(v[0], 'date') && filled(v[0], 'label') ? null : 'missing date/label on first entry',
  },
};

async function grab(tracker, resource) {
  const r = await fetch(`https://nnnsightnnn.github.io/${tracker}/api/v1/${resource}.json`, { cache: 'no-store' });
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
}

(async () => {
  const results = [];
  for (const t of TRACKERS) {
    const url = `https://nnnsightnnn.github.io/${t}/api/v1/index.json`;
    try {
      const r = await fetch(url, { cache: 'no-store' });
      let body, parsed = false, endpoints = [], commit = null, generatedAt = null;
      try {
        body = await r.json();
        parsed = true;
        endpoints = (body.endpoints || []).map(e => e.resource).sort();
        commit = body.commit;
        generatedAt = body.generatedAt || null;
      } catch {}

      const documented = (DOCUMENTED_ENDPOINTS[t] || []).slice().sort();
      const driftAdded = endpoints.filter(e => !documented.includes(e));
      const driftRemoved = documented.filter(e => !endpoints.includes(e));

      const ageHours = generatedAt
        ? Math.round((Date.now() - new Date(generatedAt).getTime()) / 36e5 * 10) / 10
        : null;
      const stale = ageHours !== null && ageHours > STALE_HOURS;

      // Field contracts — only worth running if the manifest itself parsed.
      const fieldFailures = [];
      if (parsed) {
        const allowNull = ALLOW_NULL[t] || [];
        for (const res of documented) {
          if (allowNull.includes(res)) continue;
          let payload;
          try {
            payload = await grab(t, res);
          } catch (e) {
            fieldFailures.push(`${res}: fetch failed (${e.message})`);
            continue;
          }
          const g = generic(payload);
          if (g) { fieldFailures.push(`${res}: ${g}`); continue; }
          const specific = (CONTRACTS[t] || {})[res];
          if (specific) {
            let msg = null;
            try { msg = specific(payload); }
            catch (e) { msg = `contract threw (${e.message})`; }
            if (msg) fieldFailures.push(`${res}: ${msg}`);
          }
        }
      }

      results.push({
        tracker: t, url, status: r.status, ok: r.ok && parsed, parsed,
        commit, generatedAt, ageHours, stale,
        endpoints, documented, driftAdded, driftRemoved,
        fieldFailures, hollow: fieldFailures.length > 0,
      });
    } catch (e) {
      results.push({
        tracker: t, url, status: 0, ok: false, parsed: false, error: e.message,
        stale: false, fieldFailures: [], hollow: false, driftAdded: [], driftRemoved: [],
      });
    }
  }
  console.log(JSON.stringify({ checkedAt: new Date().toISOString(), results }, null, 2));
})();
JSEOF

node ~/api/.health/check.js > ~/api/.health/new.json
cat ~/api/.health/new.json
```

Compare against the previous run *before* overwriting it, so transitions can be detected:

```bash
# prior state, for transition detection (missing on first run — treat as all-clear)
cat ~/api/.health/last.json 2>/dev/null | head -c 4000
```

Then roll `new.json` over `last.json`.

Also append a one-line summary to `~/api/.health/history.jsonl` (newline-delimited JSON) with `{checkedAt, allOk, downCount, driftCount, hollowCount, staleCount}`. Note: `.health/` is gitignored, doesn't ship to Pages — keep that ignore in place if it's not already.

### Step 3: Decide what to do

Walk through each tracker's result:

- **All four `ok: true`, zero drift, zero field failures, none stale** → only update the "Last verified" footer (Step 4a) and commit. No iMessage.
- **Any tracker `ok: false`** → flip that tracker's badge in `index.html` from `badge-live`>Live to `badge-deprecated`>Down. Append a changelog row: `<today> | <tracker> | DOWN: status=<N>, error=<short>`. Compose an iMessage (Step 5).
- **Recovery** (was previously down per `~/api/.health/last.json` from prior run, now `ok: true`) → flip badge from `badge-deprecated`>Down back to `badge-live`>Live. Append a changelog row noting recovery. No iMessage.
- **Endpoint drift** (driftAdded or driftRemoved non-empty) → append a changelog row noting drift; iMessage Kenny that the docs need a manual update. Do NOT auto-edit the endpoint table — the resource list lives in code-form on the page and Kenny should decide naming.
- **Hollow** (`hollow: true`, i.e. `fieldFailures` non-empty) → append a changelog row: `<today> | <tracker> | HOLLOW: <first 2 failures>`. Leave the badge on Live. iMessage **only if** the same tracker was not already hollow in the previous run.
- **Stale** (`stale: true`) → append a changelog row: `<today> | <tracker> | STALE: manifest <N>h old, update task may not be running`. Leave the badge on Live. iMessage **only if** it was not already stale in the previous run.
- **Hollow/stale recovery** (was hollow or stale, now clean) → append a changelog row noting it recovered. No iMessage.

Note on judgement: a tracker can be down, drifting, hollow and stale at once. Evaluate every condition, collect all of them, and still send at most one iMessage (Step 5).

### Step 4: Edit `~/api/index.html`

#### 4a. Always: update the "Last verified" footer

Find the `<footer>` block. If it contains a line `<!-- last-verified -->`, replace its content. Otherwise add it just inside the footer:

```html
<!-- last-verified --><br><small style="color: var(--dim);">Endpoints last verified: <time datetime="ISO">human-readable date</time> · all four trackers serving 200</small>
```

If not all four are OK, change "all four trackers serving 200" to "see status above".

#### 4b. Conditional: flip badges and append changelog rows

When a tracker's badge needs to change, find:
```
<h3>{Team Name} <span class="badge badge-live">Live</span></h3>
```
and swap the badge class + label per the rules above.

Append rows to the `Changelog` table (the `<tbody>` near the bottom) — newest at top, format: `<tr><td>YYYY-MM-DD</td><td>tracker name</td><td>event description</td></tr>`.

### Step 5: iMessage if needed

Use the `mcp__Read_and_Send_iMessages__send_imessage` tool to text Kenny at his own number. Body format:

```
Tracker API health check (YYYY-MM-DD)
DOWN: <tracker-1> (status=<N>)
DRIFT: <tracker-2> added=[...], removed=[...]
HOLLOW: <tracker-3> next-game: null payload
STALE: <tracker-4> manifest 41h old
Docs: https://nnnsightnnn.github.io/api/
```

Only include lines that apply. Skip iMessage entirely if there's nothing actionable — and remember that HOLLOW and STALE only count as actionable on the run where they first appear.

### Step 6: Commit and push

```bash
cd ~/api
git status --short
```

If nothing changed: skip the commit step entirely.

If something changed:
```bash
git add index.html .health/  # .health/ should be gitignored — only adds if not
git commit -m "Health check: $(date -u +%Y-%m-%d) - <one-line summary>"
git push origin main
```

Use a real one-line summary — e.g. "all four trackers green", "liverpool-tracker DOWN (504)", "hawks-tracker endpoint drift".

If a `.git/HEAD.lock` or `index.lock` blocks: try `rm -f` once; if that fails, stop and report (do not force-push).

### Step 7: Report

Return a brief summary (under 120 words):
- Pass/fail for each of the four trackers
- Any drift, hollow endpoints, or stale manifests detected
- Whether the docs page was updated (and the commit SHA if so)
- Whether an iMessage was sent

## Guardrails

- Never modify any tracker's data files — those are their own scheduled tasks' jobs. This holds especially for a hollow endpoint: report it, never populate it. Inventing a fixture or a score to make a contract pass is the worst possible outcome of this task.
- A hollow or stale tracker keeps its Live badge. Down means not responding. Do not conflate the two.
- Never edit `index.html` outside the footer "Last verified" line, the per-tracker badge, and the Changelog table.
- Do not change the URL scheme or the spec sections — those are intentional and should not drift.
- Never send more than one iMessage per run, regardless of how many things failed — concatenate into one.
- If `~/api/` doesn't exist, STOP. Don't recreate.
- Recovery transitions are silent — only the first failure pings Kenny.
