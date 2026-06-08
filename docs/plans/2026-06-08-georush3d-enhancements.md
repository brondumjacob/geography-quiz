# GeoRush 3D Enhancements Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (or
> superpowers:subagent-driven-development) to implement this plan task-by-task.

**Goal:** Add closer zoom, a higher-res satellite texture, a Manual+Auto (maptap-style
round-based) difficulty control, and the ported Supabase "play with friends" challenge mode
to `georush3d.html`.

**Architecture:** All changes are additive edits inside the single standalone
`georush3d.html`. The solo game loop, the mobile-fit fix, and the 2D `georush.html` are
preserved. Challenge mode is ported nearly verbatim from `georush.html` (same live Supabase
backend/tables), adapting only the map-layer calls to the Globe.gl API.

**Tech Stack:** Vanilla HTML/JS, Globe.gl (CDN), Supabase REST via `fetch`, Playwright for
headless verification.

**Design doc:** `docs/plans/2026-06-08-georush3d-enhancements-design.md`

## Testing notes (read first)

- There is no unit-test framework. "Tests" are assertions in the Playwright suite
  `/tmp/gr3d_test.js` (already exists, 43 passing). Run with:
  ```bash
  cd "/Users/jacobbrondum/Desktop/Claude Stuff/projects/geography-quiz" && \
    NODE_PATH=/Users/jacobbrondum/node_modules node /tmp/gr3d_test.js
  ```
- Chromium must launch with `args: ['--use-gl=swiftshader','--enable-unsafe-swiftshader','--ignore-gpu-blocklist']` for headless WebGL.
- Pages expose `window.__globeReady === true` once the globe is up, and a `window.__test`
  object for game-state hooks. Add new hooks there as needed.
- "Make it fail first" = add the assertion, run, watch it FAIL, then implement.
- Commit after each task. Do NOT push until the user asks (they will test on their phone).

---

### Task 1: Closer zoom

**Files:**
- Modify: `georush3d.html:651` (`c.minDistance = 140;`)
- Test: `/tmp/gr3d_test.js`

**Step 1: Add a failing assertion**

In `/tmp/gr3d_test.js`, after the desktop globe is ready, add a probe of the controls floor:
```js
const minDist = await page.evaluate(() => window.__test.minDistance ? window.__test.minDistance() : null);
ok('[desktop] zoom floor lowered for close-up', minDist !== null && minDist <= 110, `minDistance=${minDist}`);
```

**Step 2: Run and watch it fail**

Run the suite. Expected: FAIL — `window.__test.minDistance` is undefined (returns null), or minDistance is 140.

**Step 3: Implement**

In `georush3d.html`, change line 651:
```js
c.minDistance = 108;   // was 140 — allow zooming nearly to the surface (maptap-style)
```
Add a test hook inside the `window.__test = Object.assign(...)` block (currently
`georush3d.html:659-665`):
```js
      minDistance: () => (_globe ? _globe.controls().minDistance : null),
```

**Step 4: Run and watch it pass**

Run the suite. Expected: the new assertion PASSES; all prior assertions still pass.

**Step 5: Commit**

```bash
git add georush3d.html
git commit -m "feat(georush3d): lower zoom floor for maptap-style close-up"
```

---

### Task 2: Higher-res satellite texture with fallback

**Files:**
- Modify: `georush3d.html:644` (`.globeImageUrl(...)`) and the `initGlobe` body
- Test: `/tmp/gr3d_test.js`

**Context:** Current texture is the low-res three-globe sample
`https://unpkg.com/three-globe/example/img/earth-blue-marble.jpg`. Use a high-res Blue Marble
(~8k) that allows cross-origin use in WebGL. Recommended primary URL (verify it loads with
HTTP 200 and permissive CORS before hardcoding — run a quick `curl -I` and a Playwright probe;
if it fails, pick another reputable hi-res Blue Marble mirror):
`https://raw.githubusercontent.com/turban/webgl-earth/master/images/2_no_clouds_8k.jpg`
Define constants and an Image() preloader so a failed hi-res load falls back to the low-res URL.

**Step 1: Add a failing assertion**

In `/tmp/gr3d_test.js`:
```js
const texInfo = await page.evaluate(() => window.__test.globeTexture ? window.__test.globeTexture() : null);
ok('[desktop] hi-res texture wired with fallback', texInfo && texInfo.requested && texInfo.fallback && texInfo.requested !== texInfo.fallback,
   `tex=${JSON.stringify(texInfo)}`);
```

**Step 2: Run and watch it fail**

Run the suite. Expected: FAIL — `window.__test.globeTexture` undefined.

**Step 3: Implement**

In `georush3d.html`, just above `function initGlobe()` (near `georush3d.html:631`), add:
```js
const GLOBE_TEX_HI  = 'https://raw.githubusercontent.com/turban/webgl-earth/master/images/2_no_clouds_8k.jpg';
const GLOBE_TEX_LO  = 'https://unpkg.com/three-globe/example/img/earth-blue-marble.jpg';
let _texRequested = GLOBE_TEX_HI;
```
Change the `.globeImageUrl(...)` call (line 644) to use `GLOBE_TEX_HI`:
```js
      .globeImageUrl(GLOBE_TEX_HI)
```
After the globe is created and controls configured (right after `c.zoomSpeed = 0.8;`, before
the resize/observer block), add a fallback preloader:
```js
    // Hi-res texture can be a large download or occasionally fail; fall back to low-res.
    const _texProbe = new Image();
    _texProbe.crossOrigin = 'anonymous';
    _texProbe.onerror = () => { _texRequested = GLOBE_TEX_LO; if (_globe) _globe.globeImageUrl(GLOBE_TEX_LO); };
    _texProbe.src = GLOBE_TEX_HI;
```
Add a test hook in the `window.__test` block:
```js
      globeTexture: () => ({ requested: _texRequested, fallback: GLOBE_TEX_LO }),
```

**Step 4: Run and watch it pass**

Run the suite. Expected: new assertion PASSES; globe still mounts; no console errors;
all prior assertions pass (incl. mobile + fallback).

**Step 5: Commit**

```bash
git add georush3d.html
git commit -m "feat(georush3d): use hi-res Blue Marble texture with low-res fallback"
```

---

### Task 3: Manual + Auto (round-based) difficulty

**Files:**
- Modify: `georush3d.html` header markup (`georush3d.html:198-208`), state vars
  (`georush3d.html:747`), `evaluateDifficulty` (`georush3d.html:801-814`), `startRound`
  (`georush3d.html:865-878`), `recordRound` (`georush3d.html:896-898`), `boot`
  (`georush3d.html:994-1003`), and the `window.__test` block
- Test: `/tmp/gr3d_test.js`

**Behavior:** A `<select id="diffMode">` with `Auto · Easy · Medium · Hard` (default Auto).
- **Auto** = maptap escalation by round, looping every 10: `i = (round-1) % 10` →
  `i<4 Easy`, `i<6 Medium`, else `Hard`.
- **Manual** = lock the pool to the chosen level.
- 📈/📉 toast fires when the resolved Auto level changes between rounds.
- Replaces the rolling-average scaler (`recentKm` / `evaluateDifficulty`).

**Step 1: Add failing assertions**

In `/tmp/gr3d_test.js` (desktop, after ready):
```js
// Auto schedule, including loop past round 10
const sched = await page.evaluate(() => {
  const f = window.__test.difficultyForRound;
  return [1,4,5,6,7,10,11,15,17,20].map(r => f(r));
});
ok('[desktop] auto difficulty schedule', JSON.stringify(sched) ===
   JSON.stringify(['easy','easy','medium','medium','hard','hard','easy','medium','hard','hard']),
   `sched=${JSON.stringify(sched)}`);

// Manual lock: set Hard, advance a round, assert resolved difficulty is hard
await page.selectOption('#diffMode', 'hard');
await page.waitForTimeout(30);
const lockDiff = await page.evaluate(() => window.__test.currentDifficulty());
ok('[desktop] manual lock sets difficulty', lockDiff === 'hard', `diff=${lockDiff}`);
```

**Step 2: Run and watch it fail**

Run the suite. Expected: FAIL — `#diffMode` doesn't exist; `window.__test.difficultyForRound`
and `currentDifficulty` undefined.

**Step 3: Implement**

(a) Header markup — add the selector before `#regionFilter` (`georush3d.html:199`):
```html
      <select id="diffMode" class="gr-select" aria-label="Difficulty">
        <option value="auto">Auto</option>
        <option value="easy">Easy</option>
        <option value="medium">Medium</option>
        <option value="hard">Hard</option>
      </select>
```

(b) State var — after `let difficulty = 'easy';` (`georush3d.html:747`) add:
```js
let difficultyMode = 'auto';   // 'auto' | 'easy' | 'medium' | 'hard'
```

(c) Replace `evaluateDifficulty` (`georush3d.html:801-814`) with pure helpers:
```js
// Auto difficulty: maptap-style escalation by round, looping every 10 rounds.
function difficultyForRound(r) {
  const i = ((r - 1) % 10 + 10) % 10;   // 0-based slot, safe for r>=1
  if (i < 4) return 'easy';
  if (i < 6) return 'medium';
  return 'hard';
}
function resolveDifficulty() {
  return difficultyMode === 'auto' ? difficultyForRound(round) : difficultyMode;
}
```

(d) In `startRound` (`georush3d.html:865-878`), after `round++;` and before
`current = pickCity();`, set difficulty + fire the change toast:
```js
  const prevDiff = difficulty;
  difficulty = resolveDifficulty();
  if (difficulty !== prevDiff && round > 1) {
    const order = { easy: 0, medium: 1, hard: 2 };
    const up = order[difficulty] > order[prevDiff];
    showToast(`${up ? '📈' : '📉'} Difficulty → ${DIFF_META[difficulty].label} ${DIFF_META[difficulty].badge.slice(0, 2)}`);
  }
```

(e) In `recordRound`, remove the rolling-window scaler lines (`georush3d.html:896-898`):
```js
  recentKm.push(km);
  if (recentKm.length > 20) recentKm = recentKm.slice(-20);
  evaluateDifficulty();
```
Delete those three lines. Also delete the now-unused `let recentKm = [];`
(`georush3d.html:753`).

(f) Add a change handler and wire it in `boot` (`georush3d.html:994-1003`):
```js
function onDiffModeChange(e) {
  difficultyMode = e.target.value;
  if (gameState === 'guessing') { round--; startRound(); }  // apply immediately, like region filter
}
```
In `boot`, alongside the other listeners:
```js
  $('diffMode').addEventListener('change', onDiffModeChange);
```

(g) Test hooks — extend the `window.__test` block:
```js
      difficultyForRound: (r) => difficultyForRound(r),
      currentDifficulty: () => difficulty,
```

**Step 4: Run and watch it pass**

Run the suite. Expected: schedule + manual-lock assertions PASS; all prior pass.

**Step 5: Commit**

```bash
git add georush3d.html
git commit -m "feat(georush3d): manual + maptap-style auto round-based difficulty"
```

---

### Task 4: Challenge-mode CSS + Supabase + state scaffolding

**Files:**
- Modify: `georush3d.html` `<style>` (add challenge UI classes), `<script>` top (Supabase
  block + state vars + mode detection)
- Test: `/tmp/gr3d_test.js`

**Step 1: Add a failing assertion**

In `/tmp/gr3d_test.js`, add a small static check that the Supabase helpers and challenge CSS
exist (load the file fresh, no `?c=`):
```js
const hasSb = await page.evaluate(() => typeof window.__test.hasChallengeApi === 'function' && window.__test.hasChallengeApi());
ok('[desktop] challenge API present', hasSb === true);
```

**Step 2: Run and watch it fail**

Run the suite. Expected: FAIL — `window.__test.hasChallengeApi` undefined.

**Step 3: Implement**

(a) CSS — copy the challenge UI classes verbatim from `georush.html:155-184`
(`.gr-field`, `.gr-field label`, `.gr-input, .gr-modal select`, `.gr-rounds`,
`.gr-round-opt`, `.gr-round-opt.active`, `.gr-lb` + `td` variants, `.gr-link-box`,
`.gr-link-box code`, `.gr-err`, `.gr-muted`) into the `georush3d.html` `<style>` block,
just after the existing modal/summary styles. Skip any class already present in
`georush3d.html` (e.g. `.gr-divider`, `.gr-summary-row` — check first; add only what's
missing). Adjust `var(--wrong, #8b1a1a)` to a literal `#8b1a1a` if `--wrong` isn't defined
in `georush3d.html`'s `:root`.

(b) Supabase block — copy `georush.html:271-300` verbatim (the warning comment, `SB_URL`,
`SB_KEY`, `sbGet`, `sbPost`) and paste it at the top of the `georush3d.html` `<script>`,
right after `'use strict';` (`georush3d.html:252`).

(c) Mode detection + challenge state — after the session accumulators
(`georush3d.html:752-756`), add:
```js
const params = new URLSearchParams(window.location.search);
const challengeId = params.get('c');         // null = Solo Practice
const mode = challengeId ? 'challenge' : 'solo';

// Challenge state
let challenge = null;       // { id, cities, region, rounds, created_by }
let challengePlayer = '';
let challengeRounds = [];   // { city, km, score }
const RANK_MEDALS = ['🥇', '🥈', '🥉'];
```
Update `PLAY_URL` is already correct for 3D (`georush3d.html:759`).

(d) Test hook — in the `window.__test` block:
```js
      hasChallengeApi: () => (typeof sbGet === 'function' && typeof sbPost === 'function' && typeof createChallenge === 'function'),
```
(Note: `createChallenge` is added in Task 5; this hook will only return true after Task 5.
For Task 4, temporarily assert on `sbGet`/`sbPost` only, then tighten in Task 5. Adjust the
Task 4 assertion to check `typeof window.__test.hasChallengeApi === 'function'` and
sbGet/sbPost presence.)

Define `hasChallengeApi` for Task 4 as:
```js
      hasChallengeApi: () => (typeof sbGet === 'function' && typeof sbPost === 'function'),
```

**Step 4: Run and watch it pass**

Run the suite. Expected: challenge-API assertion PASSES; globe still mounts; all prior pass.

**Step 5: Commit**

```bash
git add georush3d.html
git commit -m "feat(georush3d): add Supabase helpers, challenge CSS and state scaffolding"
```

---

### Task 5: Challenge-mode logic + UI + globe wiring

**Files:**
- Modify: `georush3d.html` — `updateStatsBar`, `showResult`/result button label,
  `startRound`, `recordRound`, `onNext`, `openSettings`, `boot`, and add the challenge
  functions
- Test: `/tmp/gr3d_test.js`

**Context:** Port the challenge functions from `georush.html`. Adapt map calls: the 2D uses
SVG coords; the 3D uses lat/lng. The 3D already has `dropGuessPin(lat,lng)`,
`commitGuess()`, `drawResult(lat,lng)`, `clearMarkers()`, `hideMapStatus()`,
`showGlobeError()`. There is no `showMapStatus` in 3D — add a tiny one.

**Step 1: Add failing assertions**

Add a dedicated challenge block to `/tmp/gr3d_test.js` that loads the page with a stubbed
Supabase (intercept `**/rest/v1/**`) so it runs offline-deterministically:
```js
// ----- CHALLENGE MODE -----
{
  const cctx = await browser.newContext({ /* desktop */ });
  const cp = await cctx.newPage();
  // Stub Supabase: GET challenge returns a 3-city set; GET scores returns a leaderboard; POST ok.
  await cp.route('**/rest/v1/challenges**', r => r.fulfill({ status: 200, contentType: 'application/json',
    body: JSON.stringify([{ id: 'TEST01', region: 'All', rounds: 3, created_by: 'Jacob',
      cities: [{name:'Paris',country:'France',flag:'🇫🇷',lat:48.85,lon:2.35,difficulty:'easy',region:'Europe'},
               {name:'Tokyo',country:'Japan',flag:'🇯🇵',lat:35.68,lon:139.69,difficulty:'easy',region:'Asia'},
               {name:'Cairo',country:'Egypt',flag:'🇪🇬',lat:30.04,lon:31.24,difficulty:'medium',region:'Africa'}] }]) }));
  await cp.route('**/rest/v1/scores**', r => r.fulfill({ status: 200, contentType: 'application/json',
    body: JSON.stringify(r.request().method() === 'GET'
      ? [{ player_name:'Jacob', total_score: 2500, avg_distance_km: 300, best_guess_city:'Paris', best_guess_km: 50 }]
      : [{}]) }));
  await cp.goto(FILE + '?c=TEST01');
  await cp.waitForFunction(() => window.__globeReady === true, { timeout: 15000 });
  // Name prompt appears
  await cp.waitForSelector('#cpName', { timeout: 5000 });
  await cp.fill('#cpName', 'Tester');
  await cp.click('#cpStart');
  // Stats bar shows challenge round count
  const rl = await cp.locator('#roundLabel').textContent();
  ok('[challenge] round label shows N / total', /\/\s*3/.test(rl), `roundLabel="${rl}"`);
  // Play all 3 rounds via test hook + confirm
  for (let i = 0; i < 3; i++) {
    await cp.evaluate(() => window.__test.dropByLatLng(0, 0));
    await cp.click('#confirmBtn');
    await cp.click('#nextBtn');
    await cp.waitForTimeout(50);
  }
  // Leaderboard modal renders
  const lb = await cp.locator('.gr-lb').count();
  ok('[challenge] leaderboard renders', lb >= 1);
  await cctx.close();
}
```

**Step 2: Run and watch it fail**

Run the suite. Expected: FAIL — no `#cpName` prompt (challenge flow not wired).

**Step 3: Implement**

(a) `showMapStatus` helper (3D lacks it) — near `hideMapStatus`:
```js
function showMapStatus(msg) { const s = $('mapStatus'); if (s) { s.textContent = msg; s.style.display = 'flex'; } }
```

(b) `updateStatsBar` (`georush3d.html:817-822`) — branch on mode (port from
`georush.html:1077-1089`):
```js
function updateStatsBar() {
  $('roundLabel').textContent = mode === 'challenge'
    ? `Round ${round} / ${challenge ? challenge.rounds : '—'}`
    : `Round ${Math.max(1, round)}`;
  $('diffBadge').textContent = mode === 'challenge' ? '🏆 Challenge' : DIFF_META[difficulty].badge;
  const avg = round > 0 ? Math.round(sessionDistTotal / round) : null;
  $('avgLabel').textContent = avg == null ? 'Avg: — km' : `Avg: ${fmtKm(avg)} km`;
}
```

(c) Result button label — in `showResult` (find where `$('nextBtn').textContent = 'NEXT CITY →';`
is set, `georush3d.html:845`), port the challenge branch from `georush.html:1112-1117`:
```js
  const btn = $('nextBtn');
  if (mode === 'challenge' && challenge && round >= challenge.rounds) btn.textContent = 'SEE RESULTS →';
  else btn.textContent = 'NEXT CITY →';
```

(d) `startRound` — pick the challenge city when in challenge mode. Change
`current = pickCity();` (`georush3d.html:872`) to:
```js
  current = mode === 'challenge' ? challenge.cities[round - 1] : pickCity();
```
Guard the difficulty/toast block (added in Task 3) with `if (mode === 'solo')` so challenge
rounds don't run the auto schedule.

(e) `recordRound` — port the mode branch from `georush.html:1207-1213`. Since Task 3 removed
the rolling scaler, the solo branch is now empty; just record challenge rounds:
```js
  if (mode === 'challenge') {
    challengeRounds.push({ city: current.name, km, score });
  }
```
(Insert after `saveStats(s);`.)

(f) `onNext` (`georush3d.html:901-904`) — port challenge branch from `georush.html:1216-1224`:
```js
function onNext() {
  if (mode === 'challenge') {
    if (round >= challenge.rounds) { finishChallenge(); return; }
    startRound();
    return;
  }
  if (round % 10 === 0) { showSummary(); return; }
  startRound();
}
```

(g) `openSettings` (`georush3d.html:974-981`) — replace with the version that offers
"Challenge Friends" (port from `georush.html:1299-1308`):
```js
function openSettings() {
  openModal(`
    <h2>⚙️ Settings</h2>
    <p class="sub">Challenge a friend to beat your pinpointing.</p>
    <button class="gr-btn" id="openChallengeCreate">🏆 Challenge Friends</button>
    <button class="gr-btn secondary" id="closeSettings">Close</button>
  `);
  $('openChallengeCreate').addEventListener('click', showChallengeCreate);
  $('closeSettings').addEventListener('click', closeModal);
}
```

(h) Add the challenge functions — port verbatim from `georush.html`, changing only `PLAY_URL`
usage (already 3D-correct) and confirming map calls match the 3D API (they don't touch the
map; they use modals + Supabase). Port these functions exactly:
- `buildChallengeCities` (`georush.html:1352-1362`)
- `createChallenge` (`georush.html:1364-1395`)
- `initChallenge` (`georush.html:1398-1422`) — uses `showMapStatus`/`hideMapStatus` (now added)
- `promptChallengeName` (`georush.html:1424-1444`)
- `finishChallenge` (`georush.html:1446-1465`)
- `showLeaderboardLoading` (`georush.html:1467-1473`)
- `renderLeaderboard` (`georush.html:1475-1516`)
- `showChallengeCreate` (`georush.html:1310-1350`)

(i) `boot` (`georush3d.html:994-1003`) — port the mode wiring from `georush.html:1535-1549`:
```js
function boot() {
  $('nextBtn').addEventListener('click', onNext);
  $('confirmBtn').addEventListener('click', commitGuess);
  $('clearBtn').addEventListener('click', clearGuessPin);
  $('settingsBtn').addEventListener('click', openSettings);
  $('regionFilter').addEventListener('change', onRegionChange);
  $('diffMode').addEventListener('change', onDiffModeChange);

  if (mode === 'challenge') {
    // Fixed cities: hide region filter + difficulty picker.
    $('regionFilter').style.display = 'none';
    $('diffMode').style.display = 'none';
    $('settingsBtn').style.display = 'none';
  }

  initGlobe();
  if (mode === 'challenge') initChallenge();
  else startSolo();
}
```

(j) Tighten the Task 4 `hasChallengeApi` hook to include `createChallenge`:
```js
      hasChallengeApi: () => (typeof sbGet === 'function' && typeof sbPost === 'function' && typeof createChallenge === 'function'),
```

**Step 4: Run and watch it pass**

Run the suite. Expected: the challenge block PASSES (name prompt, round label "/ 3",
leaderboard renders); solo + mobile + fallback assertions still pass.

**Step 5: Commit**

```bash
git add georush3d.html
git commit -m "feat(georush3d): port Supabase challenge mode to the globe"
```

---

### Task 6: Docs + full verification

**Files:**
- Modify: `CLAUDE.md` (the `georush3d.html` section)
- Test: `/tmp/gr3d_test.js` (full run, desktop + iPhone 12 + fallback + challenge)

**Step 1: Run the full suite**

Run `/tmp/gr3d_test.js`. Expected: ALL assertions pass (prior 43 + new zoom/texture/
difficulty/challenge). Fix any regressions before proceeding.

**Step 2: Update CLAUDE.md**

In the `### georush3d.html — 3D globe practice mode` section, update:
- `minDistance` is now `108` (closer zoom).
- Texture is the hi-res Blue Marble (`GLOBE_TEX_HI`) with low-res fallback (`GLOBE_TEX_LO`);
  note the larger mobile download.
- Difficulty: `difficultyMode` ('auto'|'easy'|'medium'|'hard'), Auto = round-based maptap
  schedule (1-4 Easy, 5-6 Medium, 7-10 Hard, loops every 10) via `difficultyForRound`;
  replaces the old rolling-average scaler.
- Challenge mode + Supabase leaderboards now exist on the globe (no longer solo-only); same
  live backend/tables as `georush.html`; picker/region hidden in challenge mode.

**Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for georush3d zoom/texture/difficulty/challenge"
```

**Step 4: Final review + push decision**

Run the suite once more to confirm green, then report to the user and ask before pushing
(they want to test on their phone after deploy).

---

## Done criteria

- Zoom floor lowered; globe can be zoomed close.
- Hi-res texture loads with graceful low-res fallback.
- Difficulty: Auto follows the maptap round schedule and loops; Manual locks the level.
- Challenge mode works end-to-end on the globe (create link, play fixed cities, leaderboard).
- Full Playwright suite green on desktop + iPhone 12 + CDN-blocked fallback + challenge.
- `CLAUDE.md` updated. Nothing pushed until the user approves.
