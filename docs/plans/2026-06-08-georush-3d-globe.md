# GeoRush 3D Globe Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build `georush3d.html` — a maptap-style 3D globe practice mode that reuses GeoRush's solo-practice game logic but renders an interactive Globe.gl WebGL globe instead of the 2D SVG map.

**Architecture:** New standalone page `georush3d.html` linked from `index.html`. The parchment UI shell and all map-agnostic game logic (`CITIES`, scoring, difficulty, region filter, localStorage stats, session summary, Clear/Confirm flow, state machine) are copied from `georush.html`. The SVG/D3 map layer + hand-rolled touch handlers are replaced by a Globe.gl globe whose built-in controls handle drag-rotate and pinch-zoom, and whose `onGlobeClick(lat,lng)` drives the guess. CDN deps (Three.js/three-globe/Globe.gl + Earth textures) are allowed on this page only.

**Tech Stack:** HTML/CSS/vanilla JS, Globe.gl (UMD, CDN), Playwright for headless verification.

**Reference design:** `docs/plans/2026-06-08-georush-3d-globe-design.md`

---

## Conventions

- Reuse the exact palette, fonts, and UI classes from `georush.html`.
- Globe.gl UMD global is `Globe`. CDN: `https://cdn.jsdelivr.net/npm/globe.gl` (bundles three).
- Textures from `https://unpkg.com/three-globe/example/img/` (blue-marble, topology, night-sky).
- Country polygons from `https://unpkg.com/world-atlas/countries-110m.json` via topojson, OR reuse the inlined `window.__worldTopology` already present in `georush.html` (preferred — no extra CDN call). Decide in Task 4.
- Test harness: `/tmp/gr3d_test.js`, run with `NODE_PATH=/Users/jacobbrondum/node_modules node /tmp/gr3d_test.js` from the project dir.
- Expose minimal test hooks on `window` (e.g. `window.__test = { dropByLatLng, getState, getLayers }`) so Playwright can drive globe clicks deterministically without pixel-mapping.
- Commit after each task.

---

### Task 1: Scaffold georush3d.html from the 2D shell + game logic

**Files:**
- Create: `georush3d.html`
- Reference: `georush.html` (copy UI shell + game logic, NOT the D3/topojson/relief/SVG/touch blocks)

**Step 1:** Copy from `georush.html` into a new `georush3d.html`:
- `<head>` (Google Fonts, all `<style>` CSS including `.gr-confirm`), `<body>` markup
  (header, `#statsBar`, `.gr-map-card` with `#map`, prompt/result card with `#confirmBar`,
  `#toast`, `#overlay`).
- The game-logic JS: `$`, `calcScore`, `haversineKm`, `fmtKm`, `verdict`, `CITIES`,
  `PLAY_URL` (set to the 3d URL), `DIFF_META`, `loadStats`, `saveStats`, `showToast`,
  `evaluateDifficulty`, `updateStatsBar`, hint helpers, `renderPrompt`, `showResult`,
  `pickCity`, `startRound`, `recordRound`, `onNext`, region-filter wiring, settings/modal,
  session summary, `boot()`.
- Change the localStorage key to `georush3d_stats` so 2D and 3D stats stay separate.
- DO NOT copy: the inlined D3, topojson, `window.__worldTopology` (decide in Task 4),
  `RELIEF_URI`, `renderRelief`, `initMap`, `svgToLatLon`, `latLonToSvg`, SVG `drawResult`,
  the SVG touch/mouse handlers, `SMALL_COUNTRIES` dots. These are replaced by the globe layer.
- Strip Challenge-mode code paths (solo only): remove `?c=` handling, `challenge*`,
  Supabase `sb*` calls. Keep `mode='solo'` assumptions.
- Leave a stub `function initGlobe(){}` and call site where `initMap` was.

**Step 2: Verify it loads (no globe yet).**
Add Globe.gl `<script src="https://cdn.jsdelivr.net/npm/globe.gl"></script>` in `<head>`.
Run a quick Playwright smoke (reuse `/tmp/gr3d_test.js` skeleton): goto file, assert no
page errors, `#statsBar` present, prompt card present.
Expected: page loads, prompt shows a city, no JS errors (globe area empty).

**Step 3: Commit.**
```bash
git add georush3d.html && git commit -m "feat: scaffold georush3d shell + reused solo game logic"
```

---

### Task 2: Initialize the Globe.gl globe with textures + atmosphere

**Files:**
- Modify: `georush3d.html` (`initGlobe`)

**Step 1: Implement `initGlobe()`:**
```js
let _globe = null;
function initGlobe() {
  const el = document.getElementById('map');
  _globe = Globe()(el)
    .globeImageUrl('https://unpkg.com/three-globe/example/img/earth-blue-marble.jpg')
    .bumpImageUrl('https://unpkg.com/three-globe/example/img/earth-topology.png')
    .backgroundImageUrl('https://unpkg.com/three-globe/example/img/night-sky.png')
    .showAtmosphere(true).atmosphereColor('#bcd0d4').atmosphereAltitude(0.18)
    .width(el.clientWidth).height(el.clientHeight);
  _globe.controls().enablePan = false;
  _globe.controls().minDistance = 140;
  _globe.controls().maxDistance = 500;
  _globe.controls().autoRotate = false;
  window.addEventListener('resize', () => {
    _globe.width(el.clientWidth).height(el.clientHeight);
  });
  hideMapStatus();
}
```
Call `initGlobe()` on boot before `startRound()`.

**Step 2: Verify (Playwright).**
Assert a `<canvas>` exists inside `#map`, no console errors, and
`window.__globeReady === true` (set a flag at end of `initGlobe`).
Expected: PASS — globe canvas mounted.

**Step 3: Commit.**
```bash
git add georush3d.html && git commit -m "feat: init Globe.gl globe with textures and atmosphere"
```

---

### Task 3: Guess flow — onGlobeClick → provisional pin → Confirm/Clear

**Files:**
- Modify: `georush3d.html`

**Step 1: Implement guess functions + wire onGlobeClick.**
```js
let _pendingGuess = null;       // {lat, lng}
function setPointsAndLabels() { /* pushes _layerPoints/_layerLabels/_layerArcs to globe */ }

function dropGuessPin(lat, lng) {
  if (gameState !== 'guessing' || !current) return;
  _pendingGuess = { lat, lng };
  _globe.pointsData([{ lat, lng, kind: 'pending' }])
    .pointColor(() => '#b04a32').pointAltitude(0.02).pointRadius(0.5);
  $('confirmBar').hidden = false;
  setHint('drag to rotate • pinch to zoom • tap to move the pin');
}
function clearGuessPin() {
  _pendingGuess = null;
  _globe.pointsData([]);
  $('confirmBar').hidden = true;
  setHint('tap the globe to place your guess');
}
function commitGuess() {
  if (gameState !== 'guessing' || !_pendingGuess || !current) return;
  const { lat, lng } = _pendingGuess;
  const km = haversineKm(lat, lng, current.lat, current.lon);
  const score = calcScore(km);
  _pendingGuess = null;
  $('confirmBar').hidden = true;
  gameState = 'revealed';
  hideHint();
  drawResult(lat, lng);
  showResult(km, score);
  recordRound(km, score);
}
```
In `initGlobe`: `_globe.onGlobeClick(({lat, lng}) => dropGuessPin(lat, lng));`
Wire buttons in `boot()`: `$('confirmBtn').onclick = commitGuess; $('clearBtn').onclick = clearGuessPin;`
Add `window.__test = { dropByLatLng: dropGuessPin };`

**Step 2: Verify (Playwright).**
- `window.__test.dropByLatLng(-41.3, 174.8)` → `.gr-pending`? Instead assert
  `_globe.pointsData().length === 1` via an exposed getter and `#confirmBar` visible.
- Click `#clearBtn` → points length 0, confirmBar hidden.
Expected: PASS.

**Step 3: Commit.**
```bash
git add georush3d.html && git commit -m "feat: globe guess flow with provisional pin and confirm/clear"
```

---

### Task 4: Reveal — animated great-circle arc + answer marker + label

**Files:**
- Modify: `georush3d.html` (`drawResult`, `clearMarkers`, `startRound`)

**Step 1: Implement `drawResult` using arcs/points/labels.**
```js
function drawResult(gLat, gLng) {
  const tLat = current.lat, tLng = current.lon;
  _globe
    .pointsData([
      { lat: gLat, lng: gLng, color: '#b04a32' },
      { lat: tLat, lng: tLng, color: '#6a8c44' },
    ]).pointColor('color').pointAltitude(0.02).pointRadius(0.5)
    .arcsData([{ startLat: gLat, startLng: gLng, endLat: tLat, endLng: tLng }])
    .arcColor(() => '#5a4630').arcStroke(0.5)
    .arcDashLength(0.4).arcDashGap(0.2).arcDashAnimateTime(1500)
    .labelsData([{ lat: tLat, lng: tLng, text: current.name }])
    .labelLat('lat').labelLng('lng').labelText('text')
    .labelSize(1.2).labelColor(() => '#2a1d12').labelDotRadius(0.3);
  _globe.pointOfView({ lat: (gLat+tLat)/2, lng: (gLng+tLng)/2 }, 800);
}
function clearMarkers() {
  if (_globe) _globe.pointsData([]).arcsData([]).labelsData([]);
}
```
In `startRound()`: call `clearMarkers()`, `_pendingGuess=null`, `$('confirmBar').hidden=true`,
`setHint('tap the globe to place your guess')`, and ease camera to a neutral view:
`if (_globe) _globe.pointOfView({ lat: 20, lng: 0, altitude: 2.5 }, 600);`

**Step 2: Verify (Playwright).**
Drop a guess via `window.__test.dropByLatLng`, click `#confirmBtn`:
- `#resultView` visible, `#resultDist` matches `/km away/`.
- arcs layer length === 1, points length === 2, labels length === 1.
Click `#nextBtn` → result hidden, arcs/points/labels cleared (length 0), new prompt.
Expected: PASS.

**Step 3: Commit.**
```bash
git add georush3d.html && git commit -m "feat: globe reveal with animated arc, answer marker, label"
```

---

### Task 5: Country borders for crispness

**Files:**
- Modify: `georush3d.html`

**Step 1:** Add country outline polygons. Prefer reusing inlined topology if cheap;
otherwise load `https://unpkg.com/world-atlas/countries-110m.json` and convert with the
topojson already includable, OR fetch GeoJSON. Implementation:
```js
fetch('https://unpkg.com/world-atlas/countries-110m.json')
  .then(r => r.json())
  .then(topo => {
    const feats = topojson.feature(topo, topo.objects.countries).features;
    _globe.polygonsData(feats)
      .polygonCapColor(() => 'rgba(0,0,0,0)')
      .polygonSideColor(() => 'rgba(0,0,0,0)')
      .polygonStrokeColor(() => 'rgba(90,70,48,0.55)')
      .polygonAltitude(0.005);
  }).catch(() => {/* textures alone remain */});
```
(If using this, include the topojson-client inline block from `georush.html`, or use
world-atlas pre-converted GeoJSON to avoid topojson.) Graceful failure = no borders.

**Step 2: Verify (Playwright).**
Assert no new console errors and `_globe.polygonsData().length > 0` once loaded (poll).
Expected: PASS (or borders skipped gracefully if offline in CI — then assert no crash).

**Step 3: Commit.**
```bash
git add georush3d.html && git commit -m "feat: add country border polygons to globe"
```

---

### Task 6: Graceful WebGL/CDN failure fallback

**Files:**
- Modify: `georush3d.html` (`initGlobe`, `#mapStatus`)

**Step 1:** Wrap `initGlobe` in try/catch; if `Globe` is undefined or WebGL unsupported,
show `#mapStatus` with a message + link to `georush.html`:
```js
if (typeof Globe === 'undefined') {
  showMapStatus('3D globe failed to load. Try the 2D version → georush.html');
  return;
}
```
Make the message a real link.

**Step 2: Verify (Playwright).**
Run page with Globe forced undefined (route-block the CDN) → assert `#mapStatus` visible
with the fallback text, no crash.
Expected: PASS.

**Step 3: Commit.**
```bash
git add georush3d.html && git commit -m "feat: graceful fallback when globe/CDN unavailable"
```

---

### Task 7: Link from index.html

**Files:**
- Modify: `index.html`

**Step 1:** Add a parchment card linking to `georush3d.html` ("🌐 GeoRush 3D —
globe practice"), matching existing card styling. Read `index.html` first.

**Step 2: Verify (Playwright).** Load `index.html`, assert an anchor with
`href="georush3d.html"` exists and is visible.

**Step 3: Commit.**
```bash
git add index.html && git commit -m "feat: link GeoRush 3D from home page"
```

---

### Task 8: Full end-to-end verification + mobile viewport

**Files:**
- Create: `/tmp/gr3d_test.js` (final consolidated suite)

**Step 1:** Consolidated Playwright suite asserting the full loop (desktop + a mobile
emulation context with `hasTouch:true`, iPhone-ish viewport):
load → no errors → globe canvas → drop guess → confirm → result/arc → next → repeat,
plus Clear, plus fallback. Run both viewports.

**Step 2: Run.**
`NODE_PATH=/Users/jacobbrondum/node_modules node /tmp/gr3d_test.js`
Expected: all PASS, 0 failed.

**Step 3:** Update `CLAUDE.md`: add `georush3d.html` to the file list, note the CDN
exception to Rule 1, and the separate `georush3d_stats` localStorage key. Commit.
```bash
git add georush3d.html index.html CLAUDE.md && git commit -m "test: e2e verify georush3d; document 3D mode"
```

---

## Done criteria
- `georush3d.html` plays a full solo round on a detailed 3D globe: clue → spin/zoom →
  tap → Confirm → distance/score/arc → Next, with difficulty scaling, region filter,
  stats, and session summary working.
- Pinch-zoom works via Globe.gl built-in controls (no hand-rolled touch).
- Graceful fallback to 2D if WebGL/CDN unavailable.
- Linked from `index.html`. All Playwright checks pass on desktop + mobile viewport.
- 2D `georush.html` untouched and still fully self-contained.
