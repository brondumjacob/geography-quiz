# GeoRush — City Pinpointing Practice Game

## Project
App name: GeoRush
Repo: https://github.com/brondumjacob/geography-quiz
Live at: https://brondumjacob.github.io/geography-quiz/
Local path: /Users/jacobbrondum/Desktop/Claude Stuff/projects/geography-quiz

Files in repo root:
- `africa.html`  — Country quiz, Africa — DO NOT TOUCH
- `world.html`   — Country quiz, World  — DO NOT TOUCH
- `georush.html` — [BUILT] GeoRush city pinpointing game
- `georush3d.html` — [BUILT] GeoRush 3D — globe practice mode (Globe.gl, solo + challenge mode)
- `index.html`   — [BUILT] Home page linking all three games

---

## Implementation Notes / Current State

Deviations from / additions to the original spec below, kept in sync as work lands:

- **Palette**: matches `world.html` exactly (parchment + earthy tones), not the Rule 5
  hex values in this doc — those were stale. world.html is the source of truth.
- **Borderless map**: country border strokes removed from land paths and island
  dots (maptap-style flat fill). Ocean stays flat `#bcd0d4`.
- **Inline hypsometric relief**: a public-domain Natural Earth 1:50m cross-blended
  hypsometric raster (equirectangular, ~440 KB base64) is embedded as a `const
  RELIEF_URI` data URI. At map init it is reprojected pixel-by-pixel into the active
  `geoNaturalEarth1` projection via `projection.invert` on a canvas, clipped to land,
  and drawn as the map background — so elevation tints align with the vector land and
  the ocean stays flat. Solid land fill remains underneath as a fallback if the image
  fails to load. Fully self-contained: NO runtime fetch. File is now ~900 KB.
  - To regenerate: download `HYP_50M_SR_W` from naciscdn.org, `sips -Z 2048 -s format
    jpeg`, base64-encode, replace the `RELIEF_URI` constant.
- **Map navigation**: mouse wheel-zoom and click-drag-pan are enabled in BOTH the
  `guessing` and `revealed` states so you can zoom/pan to place a precise guess.
  A stationary click drops/moves the guess pin; a drag does not (guarded by `_mMoved`,
  which is intentionally NOT reset on mouseup so the trailing `click` can tell a drag
  from a click). `startRound()` calls `resetZoom()` so each new city starts at the full
  world view.
- **Zoom-transform coordinate fix**: the map group `_g` carries the d3 zoom transform,
  so a click's viewport coords MUST be run through `_getT().invert(...)` (via the
  `clientToMap()` helper) before `svgToLatLon()` / drawing — otherwise a click while
  zoomed lands at a completely different lat/lon (e.g. clicking NZ scored as Africa).
  At zoom=1 the transform is identity, which is why the bug only showed when zoomed.
- **MapTap-style confirm flow**: a tap/click no longer commits instantly. It drops a
  *provisional* pin (`.gr-pending`, in map/group coords) and reveals a Clear/Confirm
  bar (`#confirmBar`). The player can pan/pinch-zoom and re-tap to refine; the guess is
  only scored on **Confirm** (`commitGuess()`). Clear (`clearGuessPin()`) removes the
  pin. `dropGuessPin()` sets `_pendingGuess`; `startRound()` resets it and hides the bar.
- **Touch (iOS-safe) gestures while guessing**: touch handlers now run in BOTH `guessing`
  and `revealed`. One finger pans, two fingers pinch-zoom, in both states. A clean
  single-finger tap (tracked via `_tap` with a `TAP_SLOP` of 10px; cleared on any
  2-finger gesture) drops/moves the provisional pin on `touchend`. This fixes the bug
  where the first finger of a two-finger pinch instantly committed a guess on iPhone.
  Rule 3's iOS constraints are preserved: `touch-action:none`, all touch handlers with
  `{passive:false}`, and `e.preventDefault()` in `touchstart`.

### `georush3d.html` — 3D globe practice mode

- Standalone 3D globe version that **reuses `georush.html`'s solo game logic** (CITIES
  database, scoring, haversine, difficulty control (manual + round-based auto), region filter, localStorage,
  session summary, Clear/Confirm flow) but renders an interactive **Globe.gl WebGL globe**
  instead of the 2D SVG map.
- **CDN exception to Rule 1**: this page (and ONLY this page) loads external deps at
  runtime — Globe.gl (`cdn.jsdelivr.net/npm/globe.gl`), a hi-res 4k Blue Marble day
  texture (`raw.githubusercontent.com/turban/webgl-earth/.../2_no_clouds_4k.jpg`,
  CORS-enabled) with a graceful fallback to the low-res
  `unpkg.com/three-globe/example/img/earth-blue-marble.jpg` if it fails to load, plus the
  topology bump and night-sky background textures from `unpkg.com/three-globe`. Hi-res is a
  larger download → slightly slower first paint on mobile. The 2D
  `georush.html` remains fully self-contained. (Country-border polygons were removed per
  request — the textured globe has no vector outlines.)
- **Globe sizing / click target**: the neutral view altitude is `GLOBE_VIEW_ALT = 1.45`
  (and `controls.maxDistance = 360`) so the globe nearly fills the map card. This matters
  for clicking: `onGlobeClick` only fires when the ray hits the globe sphere, so a small/far
  globe (the old `altitude: 2.5`) left most clicks landing in empty space and doing nothing.
  `controls.minDistance` was lowered from 140 to `108` (globe radius is 100, so the camera
  can approach to ~0.08 altitude) for maptap-style close-up zoom and more precise guesses.
- **Mobile width fix**: the map card uses `aspect-ratio: 1.7/1` + `min-height: 300px`. On
  narrow screens the aspect-derived height fell below 300px, so the browser back-computed
  card *width* from the min-height (300 × 1.7 = 510px), overflowing the viewport. Fixed by
  giving `.gr-map-card` an explicit `width: 100%` (width becomes the definite dimension, so
  aspect-ratio derives height from it, not vice-versa). `#map` is now `position:absolute;
  inset:0` so the Globe.gl canvas can't push the container wider, and a `ResizeObserver` on
  `#map` keeps the globe sized to its real container after layout/fonts settle (the old
  `window.resize`-only listener never fired on initial settle). `overflow-x:hidden` on
  `html,body` is a final guard.
- **Separate localStorage key**: `georush3d_stats` — 2D and 3D stats are independent.
- **Difficulty (solo)**: a header picker offers **Auto · Easy · Medium · Hard** (default
  Auto). Auto escalates by round number, maptap-style, looping every 10 rounds —
  `difficultyForRound(r)` maps `((r-1)%10)` to rounds 1–4 Easy, 5–6 Medium, 7–10 Hard, then
  repeats. This REPLACED the old rolling-average auto-scaler. Manual locks the pool to one
  level. The 📈/📉 toast fires when the Auto level changes between rounds.
- **Challenge mode (ported from 2D)**: `georush3d.html?c=ID` plays a fixed Supabase-backed
  city set on the globe with a live leaderboard — same `challenges`/`scores` tables, same
  live anon key, and the same `generateChallengeId`/`sbGet`/`sbPost`/`createChallenge`/
  `initChallenge`/`renderLeaderboard` flow as `georush.html` (ported verbatim; only the
  drop-pin / draw-arc calls use the globe API). In challenge mode the region filter and
  difficulty picker are hidden and difficulty is fixed to the creator's set. `initChallenge`
  clamps `challenge.rounds` to the number of available cities so a malformed Supabase row
  can't index past the city array.
- **Graceful fallback**: if WebGL is unavailable or the Globe.gl CDN fails, `#mapStatus`
  shows a message linking to the 2D `georush.html`; the game loop never crashes.
- Linked from `index.html` via a "GeoRush 3D" featured card.

---

## Supabase Config

Use these exact placeholder strings in georush.html.
The user will replace them manually before deploying.

  SUPABASE_URL_HERE
  SUPABASE_ANON_KEY_HERE

Add this comment directly above them in georush.html:
```js
// ⚠️  Replace SUPABASE_URL_HERE and SUPABASE_ANON_KEY_HERE
//     with your actual Supabase credentials before deploying.
//     The anon key is safe to be public — security is enforced via RLS.
const SB_URL = 'SUPABASE_URL_HERE';
const SB_KEY = 'SUPABASE_ANON_KEY_HERE';
```

---

## Two Play Modes

### Mode 1 — Solo Practice (default, no URL param)
- Endless rounds, no daily limit — pure practice
- Auto-scaling difficulty based on rolling performance window
- Region filter: All / Africa / Europe / Americas / Asia / Oceania
- Personal stats tracked in localStorage
- Session summary + shareable score card every 10 rounds

### Mode 2 — Challenge Round (URL param: ?c=CHALLENGE_ID)
- One person creates a challenge: picks region, round count, enters nickname
- App generates a fixed city set, saves to Supabase, returns a unique ID
- Shareable URL: `georush.html?c=A3X7KQ`
- Anyone who clicks the link plays the exact same cities in the same order
- Challenge creator plays blind — they do not preview the cities
- After completing: score saved to Supabase, leaderboard shown immediately
- Leaderboard updates live as more friends complete the challenge

On page load, detect mode:
```js
const params = new URLSearchParams(window.location.search);
const challengeId = params.get('c'); // null = Solo Practice
```

---

## Core Gameplay Loop

1. Prompt shown: "Find:  NAIROBI  🇰🇪"
2. Player taps/clicks anywhere on the map to place a guess pin
3. Result: animated dashed line from pin to correct location, distance in km, score
4. "Next City →" button advances to next round
5. Repeat indefinitely (Solo) or until rounds complete (Challenge)

---

## Scoring

```js
const MAX_DIST_KM = 10000;
function calcScore(distanceKm) {
  return Math.round(1000 * Math.max(0, 1 - distanceKm / MAX_DIST_KM));
}
// 0 km    = 1000 pts
// 1000 km =  900 pts
// 5000 km =  500 pts
// 10000 km =  0 pts
```

---

## Haversine Distance

```js
function haversineKm(lat1, lon1, lat2, lon2) {
  const R = 6371;
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLon = (lon2 - lon1) * Math.PI / 180;
  const a = Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
    Math.sin(dLon / 2) ** 2;
  return R * 2 * Math.asin(Math.sqrt(a));
}
```

---

## Auto-Scaling Difficulty (Solo Practice only)

Start at 'easy'. Evaluate after every guess once 5+ guesses have been made.
Use a rolling window of the last 5 guesses.

Upgrade rules:
  easy   → medium : rolling avg < 400 km
  medium → hard   : rolling avg < 600 km

Downgrade rules:
  medium → easy   : rolling avg > 2500 km
  hard   → medium : rolling avg > 2000 km

Show current difficulty as a badge in the stats bar:
  🟢 Easy   🟡 Medium   🔴 Hard

On change: show a 2-second toast:
  "📈 Difficulty → Medium 🟡"
  "📉 Difficulty → Easy 🟢"

In Challenge Mode: difficulty is fixed to what the creator chose.
Auto-scaling only applies in Solo Practice.

---

## City Database

Inline ~350 cities as a JS const array. Each entry:

```js
{ name: 'Nairobi', country: 'Kenya', flag: '🇰🇪',
  lat: -1.286, lon: 36.817,
  difficulty: 'easy', region: 'Africa' }
```

Target distribution:
  Easy   (~120): World capitals + mega-cities
  Medium (~120): Major regional cities, secondary capitals
  Hard   (~110): Smaller, less obvious cities

Cover all 5 regions (Africa, Europe, Americas, Asia, Oceania) roughly
equally at every difficulty level.
When a region filter is active, draw only from that region.
Auto-scaling still applies within the active region filter.

---

## UI Layout — Mobile First

Default state:
```
┌─────────────────────────────────────────┐
│  🌍 GeoRush       [Region ▾]    [⚙️]   │
├─────────────────────────────────────────┤
│  Round 7  •  🟡 Medium  •  Avg: 843 km │
├─────────────────────────────────────────┤
│                                         │
│          [ WORLD MAP SVG ]              │
│    tap anywhere to place your guess     │
│                                         │
├─────────────────────────────────────────┤
│  🔍  Find:  NAIROBI   🇰🇪   Kenya      │
└─────────────────────────────────────────┘
```

After guess:
```
├─────────────────────────────────────────┤
│  🎯  234 km away  •  Score: 976 / 1000  │
│           [ NEXT CITY → ]               │
└─────────────────────────────────────────┘
```

Post-guess map state:
- Red pin 📍 at guess location
- Green dot ✅ at correct city with name label
- Animated dashed line drawn over ~600ms connecting the two
- Emoji result indicator:
    🎯 < 200 km   "Incredible!"
    ✅ < 800 km   "Close!"
    📍 < 2500 km  "Not bad"
    ❌ > 2500 km  "Way off"

Challenge leaderboard (Mode 2, shown after final round):
```
🏆 GeoRush Challenge  •  10 European Cities
─────────────────────────────────────────────
  🥇  Jacob     🎯  avg  234 km    8,120 pts
  🥈  Sarah     📍  avg  891 km    5,450 pts
  🥉  Mike      ❌  avg 2,340 km   2,100 pts
─────────────────────────────────────────────
      [ Share My Score ]   [ Create New Challenge ]
```

Create Challenge UI (⚙️ → Challenge Friends):
```
  Your name:  [ text input           ]
  Region:     [ All          ▾       ]
  Rounds:     [ 5  ]  [ 10  ]  [ 20 ]
              [ Create Challenge →   ]
```

On success:
```
  ✅ Challenge created!
  georush.html?c=A3X7KQ
  [ Copy Link ]
```

Solo session summary (every 10 rounds, Mode 1 only):
```
🌍 GeoRush — 10 Rounds
────────────────────────
🟡 Difficulty reached: Medium
🎯 Best guess: Tokyo — 12 km
📍 Avg distance: 843 km
🏆 Score: 8,430 / 10,000
        [ Copy Score ]
```

Copy Score text:
```
🌍 GeoRush — 10 rounds
🟡 Medium difficulty
🎯 Best: 12 km (Tokyo)
📍 Avg: 843 km away
🏆 8,430 / 10,000
▶ Play: https://brondumjacob.github.io/geography-quiz/georush.html
```

---

## Supabase Integration

Use native fetch — do NOT import the Supabase JS SDK.

```js
async function sbGet(path) {
  try {
    const r = await fetch(`${SB_URL}/rest/v1/${path}`, {
      headers: { apikey: SB_KEY, Authorization: `Bearer ${SB_KEY}` }
    });
    return r.ok ? r.json() : null;
  } catch { return null; }
}

async function sbPost(path, body) {
  try {
    const r = await fetch(`${SB_URL}/rest/v1/${path}`, {
      method: 'POST',
      headers: {
        apikey: SB_KEY,
        Authorization: `Bearer ${SB_KEY}`,
        'Content-Type': 'application/json',
        Prefer: 'return=representation'
      },
      body: JSON.stringify(body)
    });
    return r.ok ? r.json() : null;
  } catch { return null; }
}
```

All Supabase calls must handle null returns gracefully — show a
user-facing error message, never crash the game.

The 4 API calls:
```js
// Create a challenge
await sbPost('challenges', { id, cities, region, rounds, created_by });

// Load a challenge
await sbGet(`challenges?id=eq.${id}&select=*`);

// Submit a score
await sbPost('scores', {
  challenge_id, player_name, avg_distance_km,
  total_score, best_guess_city, best_guess_km, rounds_played
});

// Get leaderboard for a challenge
await sbGet(`scores?challenge_id=eq.${id}&order=total_score.desc&select=*`);
```

Challenge ID generation:
```js
function generateChallengeId() {
  return Math.random().toString(36).slice(2, 8).toUpperCase();
}
```

---

## MANDATORY: Existing Code Patterns

Read world.html fully before writing any code for georush.html.
These patterns exist because of iOS Safari compatibility requirements.
Deviating will break GeoRush on iPhone.

### Rule 1 — Self-contained file, zero external requests at runtime

Copy these three blocks verbatim from world.html into georush.html:
- D3 v7.9.0 inline script block (~280 KB)
- topojson-client v3.1.0 inline script block
- window.__worldTopology = {...} inline script block

Allowed external requests at runtime:
  1. Google Fonts CSS <link> in <head>
  2. Supabase REST API calls via sbGet / sbPost

No other external requests. No CDN fetches. No dynamic script loading.

### Rule 2 — Map projection

```js
const projection = d3.geoNaturalEarth1()
  .fitExtent([[8, 8], [mapW - 8, mapH - 8]],
    { type: 'FeatureCollection', features });
const pathGen = d3.geoPath().projection(projection);
_projection = projection; // stored module-level
```

Coordinate conversion:
```js
function svgToLatLon(svgX, svgY) {
  const [lon, lat] = _projection.invert([svgX, svgY]);
  return { lat, lon };
}
function latLonToSvg(lat, lon) {
  return _projection([lon, lat]); // [svgX, svgY]
}
```

### Rule 3 — Touch Events for iOS Safari (NON-NEGOTIABLE)

Use raw Touch Events on the map div with { passive: false }.
Do NOT use Pointer Events. Do NOT attach d3.zoom() for gesture handling.
This is the only approach that works reliably on iOS Safari.

Before guess is placed — any tap = guess pin, no panning:
```js
mapDiv.addEventListener('touchstart', e => {
  e.preventDefault();
  if (gameState !== 'guessing') return;
  const touch = e.touches[0];
  const rect = mapDiv.getBoundingClientRect();
  const svgX = (touch.clientX - rect.left) * (_mapW / rect.width);
  const svgY = (touch.clientY - rect.top)  * (_mapH / rect.height);
  placeGuess(svgX, svgY);
}, { passive: false });
```

Mouse equivalent for desktop:
```js
svgEl.addEventListener('click', e => {
  if (gameState !== 'guessing') return;
  const rect = svgEl.getBoundingClientRect();
  const svgX = (e.clientX - rect.left) * (_mapW / rect.width);
  const svgY = (e.clientY - rect.top)  * (_mapH / rect.height);
  placeGuess(svgX, svgY);
});
```

After guess is placed (result shown, before Next pressed):
Allow pan and zoom. Use same _zoom.transform pattern from world.html —
d3.zoom() for transform state only, Touch Events for gesture handling.

### Rule 4 — Small island dot markers

Copy the SMALL_COUNTRIES object and dot-drawing code verbatim from world.html.
These make Pacific islands, Caribbean nations, etc. visible and tappable.
Without them, many cities in the database will be invisible on the map.

### Rule 5 — Visual design (match existing quizzes exactly)

```css
:root {
  --bg:      #faf2dd;
  --surface: #f0e6c8;
  --line:    #5a4630;
  --text:    #2a1d12;
  --accent:  #b8450a;
  --correct: #2d6a2d;
  --wrong:   #8b1a1a;
}
```

Fonts: 'Fraunces' (serif) + 'Manrope' (sans) — Google Fonts.
Map: ocean #bcd0d4, land #cdb78a, borders #5a4630 at 0.4px.

---

## localStorage Schema

```js
const STORAGE_KEY = 'georush_stats';
{
  totalRounds: 0,
  totalScore: 0,
  totalDistanceKm: 0,
  bestGuessKm: Infinity,
  bestGuessCity: '',
  bestSessionAvgKm: Infinity,
  history: []  // last 50: { city, guessKm, score, ts }
}
```

---

## Game State Machine

States:
  'loading'             — page loading
  'idle'                — map ready, first city about to show
  'guessing'            — waiting for map tap/click
  'revealed'            — result shown, waiting for Next
  'summary'             — session summary overlay (every 10 rounds, Solo)
  'challenge_create'    — create challenge overlay
  'challenge_complete'  — all challenge rounds done, leaderboard shown

Transitions:
  loading           → idle                (map initialized)
  idle              → guessing            (auto)
  guessing          → revealed            (player taps map)
  revealed          → guessing            (Next clicked)
  revealed          → summary             (10th round, Solo only)
  summary           → guessing            (dismissed)
  revealed          → challenge_complete  (final round, Challenge mode)
  challenge_complete → challenge_create   (Create New Challenge clicked)

---

## index.html — Navigation Home Page

Build this after georush.html is complete and working.

Layout:
```
┌─────────────────────────────────────────┐
│         🌍  Geography Practice          │
│      sharpen your geography skills      │
├─────────────────────────────────────────┤
│  ┌───────────────────────────────────┐  │
│  │  🌍 GeoRush           FEATURED   │  │  ← larger card
│  │  City pinpointing practice game   │  │
│  │  [ Play GeoRush → ]               │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌────────────────┐  ┌────────────────┐ │
│  │ 🗺 Africa Quiz  │  │ 🌐 World Quiz  │ │  ← two smaller cards
│  │ 54 countries   │  │ 195 countries  │ │
│  │  [ Play → ]    │  │  [ Play → ]    │ │
│  └────────────────┘  └────────────────┘ │
└─────────────────────────────────────────┘
```

- Same visual design as the quizzes (parchment, Fraunces + Manrope, earthy palette)
- GeoRush card is visually dominant — it's the featured game
- Mobile-first, single column on small screens
- No external dependencies except Google Fonts
- Plain HTML/CSS — no JS needed

---

## Build Order — Follow This Exactly

1.  Copy inline D3 + topojson-client + world topology from world.html (verbatim)
2.  Copy SMALL_COUNTRIES + dot-drawing code from world.html (verbatim)
3.  Build city database (~350 cities, inline JS const array)
4.  Build game state machine and gameState variable
5.  Build map: country fill rendering, placeGuess(), click/touch handlers
6.  Build result display: pins, animated dashed line, distance, score, emoji
7.  Build auto-scaling difficulty with toast notifications
8.  Build region filter dropdown
9.  Build localStorage stats + solo session summary + Copy Score button
10. Build Challenge Mode:
      - URL param detection on load (?c=)
      - Create challenge UI + sbPost to Supabase
      - Load challenge via sbGet + render city set
      - Submit score + fetch and display leaderboard
11. Verify SUPABASE_URL_HERE and SUPABASE_ANON_KEY_HERE are placeholders
    with the warning comment above them
12. Build index.html — the home page (spec above)
