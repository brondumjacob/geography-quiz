# GeoRush 3D Globe Practice Mode — Design

> Date: 2026-06-08 • Status: Approved • Target file: `georush3d.html`

## Purpose

Build a maptap-style 3D globe version of GeoRush — "essentially maptap but with the
ability to practice." A new standalone page that reuses GeoRush's existing solo-practice
game logic but renders the map as an interactive 3D WebGL globe instead of a flat SVG map.

## Scope (this build)

**In:** Full solo practice on a 3D globe — detailed textured globe, clue prompt,
spin/zoom, tap-to-guess, Clear/Confirm flow, distance + score, animated great-circle arc
to the answer, plus the existing city database, auto-scaling difficulty, region filter,
localStorage stats, and the every-10-rounds session summary.

**Out (deferred):** Challenge mode + Supabase leaderboards on the globe (the 2D
`georush.html` keeps these). Can be ported later.

## Key Decisions

- **Separate page** `georush3d.html`, linked from `index.html`. The working 2D
  `georush.html` stays intact as a fallback. One page's bugs can't break the other.
- **Globe.gl** (wrapper over Three.js + three-globe) for rendering. Chosen over raw
  Three.js because it provides a textured/atmosphere globe, country-border polygons,
  animated arcs, point/label markers, built-in drag-rotate + pinch/wheel-zoom controls,
  and an `onGlobeClick(lat, lng)` callback. The built-in Three.js/OrbitControls multi-touch
  handling avoids the hand-rolled iOS touch pitfalls that affected the 2D version.
- **CDN dependencies allowed on this page only.** Rule 1 (zero external requests,
  fully self-contained) is relaxed for `georush3d.html`: Three.js, three-globe, Globe.gl,
  and Earth textures load from CDN. The 2D `georush.html` remains fully self-contained.
  Note this exception in CLAUDE.md.

## Architecture

`georush3d.html` is one standalone HTML file with the same parchment UI shell
(header, stats bar, map card, prompt/result card, toast, modal), fonts, and palette
as `georush.html`.

### Reused verbatim from georush.html (game logic, map-agnostic)
- `CITIES` database (~350 cities)
- `calcScore`, `haversineKm`, `verdict`, `fmtKm`
- `evaluateDifficulty`, `DIFF_META`, difficulty toasts
- `loadStats` / `saveStats` / `updateStatsBar`, localStorage schema
- Session summary (every 10 rounds) + Copy Score
- Region filter dropdown + `pickCity`
- Game state machine: `startRound`, `recordRound`, `showResult`, `renderPrompt`, `onNext`
- Prompt/result card markup, toast, modal, Clear/Confirm bar

### New 3D map layer (replaces SVG/D3 + touch handlers)
- `initGlobe()` — create Globe.gl instance, textures, atmosphere, country polygons, controls
- `latLonToGuess` via `onGlobeClick(lat, lng)` — Globe.gl gives lat/lng directly
- `dropGuessPin(lat, lng)` — provisional marker (Globe.gl points/custom layer) + show Confirm bar
- `clearGuessPin()` — remove provisional marker, hide Confirm bar
- `commitGuess()` — score via haversine, reveal: animated arc + answer marker + label
- `drawResult` — great-circle `arcsData` from guess → answer, green target point + city label
- `clearMarkers()` — clear points/arcs/labels between rounds
- `resetView()` — ease camera to a neutral globe view on each new round

### CDN deps (georush3d.html only)
- Three.js, three-globe, Globe.gl
- Earth day texture (high-res Blue Marble), topology/bump map, night-sky background
- World-atlas countries GeoJSON for crisp border polygons

## Interaction / Data Flow

1. `startRound()` picks a city, shows the clue (`Find: WELLINGTON 🇳🇿`), calls `resetView()`.
2. Player drags to rotate, pinch/wheel to zoom. A tap → `onGlobeClick(lat,lng)` →
   `dropGuessPin` drops a provisional marker and shows the Clear/Confirm bar. Tap again
   to move it.
3. **Confirm** → `commitGuess`: `haversineKm(guess, city)` → `calcScore` → animated
   great-circle arc from guess to the true location, green marker + city label, and the
   result card shows distance / score / emoji verdict.
4. **Next City →** advances; stats, auto-difficulty, region filter, and session summary
   all behave exactly as in the 2D version.

## Detail / Textures

Globe.gl Earth with a high-res day texture (Blue Marble), a topology bump map for terrain
relief, a starfield background, and atmosphere glow. Country borders rendered as polygon
outlines from a CDN world-atlas GeoJSON so boundaries stay crisp at any zoom. Pick the
highest-resolution textures that load reliably from CDN.

## Error Handling

- If WebGL is unavailable or a CDN asset fails to load, show a friendly message in the
  map area with a link back to the 2D version. Never crash the game loop.
- Textures load progressively; the globe is interactive as soon as controls are ready.

## Testing

Playwright headless run asserting:
- Globe canvas mounts; no console/page errors on load.
- Simulated globe click drops a provisional pin and shows the Confirm bar.
- Confirm reveals a distance/score in the result card; arc layer has data.
- Clear removes the provisional pin and hides the Confirm bar.
- Next City advances the round (result hidden, new clue shown).
- Re-run on a mobile-emulation viewport for layout sanity.

## index.html

Add a card linking to `georush3d.html` (e.g., "GeoRush 3D — globe practice"),
matching the existing parchment card styling.

## Out of Scope / Follow-ups

- Challenge mode + Supabase leaderboards on the globe.
- Inlining deps for full offline support (would bloat the file to several MB).
