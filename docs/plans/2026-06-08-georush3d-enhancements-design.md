# GeoRush 3D Enhancements — Design

> Date: 2026-06-08 • Status: Approved • Target file: `georush3d.html`

## Purpose

Bring the 3D globe practice mode closer to maptap and to feature parity with the 2D
game: let players zoom in for precise guesses, sharpen the map, give explicit control over
difficulty (plus a maptap-style auto progression), and add the "play with friends" challenge
mode that already exists in the 2D version.

The 2D `georush.html` stays untouched as a working fallback. All four enhancements are
additive changes inside `georush3d.html` and must preserve the existing solo game loop and
the recent mobile-fit fix.

## Scope

**In:**
1. Closer zoom on the globe (maptap-like).
2. Higher-resolution satellite (Blue Marble) texture with graceful fallback.
3. Difficulty: a Manual selector (Easy/Medium/Hard) plus an Auto mode that escalates by
   round number, maptap-style.
4. Port the full Supabase challenge ("play with friends") flow from `georush.html`.

**Out (deferred):** new map styles beyond hi-res satellite, country borders (stay removed),
any Supabase schema changes (reuse the live tables as-is).

## 1. Closer zoom

Lower the OrbitControls zoom floor so the camera can approach the surface like maptap.

- `controls.minDistance`: `140` → `~108` (globe radius is 100; this allows altitude ~0.08).
- Keep `controls.maxDistance = 360` (the far view that keeps clicks landing on the globe).
- No change to click handling: `onGlobeClick` still fires when zoomed in. Closer zoom
  directly improves guess precision, which is also what makes the Auto-difficulty story work.

## 2. Higher-res satellite texture

Swap the low-res `earth-blue-marble.jpg` (~2048px) for a high-resolution Blue Marble texture
(target ~8k) from a reliable CDN, keeping the realistic look but staying sharp when zoomed.

- **Graceful fallback:** if the hi-res texture fails to load, fall back to the current
  low-res `unpkg.com/three-globe/example/img/earth-blue-marble.jpg` so the globe never
  renders blank. The globe stays interactive while the texture loads.
- Note the tradeoff in CLAUDE.md: hi-res is a larger download → slower first paint on mobile.
- This is the existing CDN exception to Rule 1; no change to that policy.

## 3. Difficulty: Manual + Auto

Add a difficulty control in the header/stats area with options: **Auto · Easy · Medium · Hard**.
Default is **Auto**.

- **Auto** = maptap-style escalation by round number, looping every 10 rounds. Within each
  block of 10, derived from the round counter:
  - rounds 1–4 → Easy
  - rounds 5–6 → Medium
  - rounds 7–10 → Hard
  - then repeats (11–14 Easy, 15–16 Medium, 17–20 Hard, …).
  - Implementation: `((round - 1) % 10)` → 0–3 Easy, 4–5 Medium, 6–9 Hard.
  - This REPLACES the current rolling-average auto-scaler (`evaluateDifficulty`).
- **Manual (Easy/Medium/Hard)** = locks the active city pool to that level; no auto changes.
- The existing 📈/📉 toast fires when the Auto level changes between rounds.
- `pickCity()` continues to filter `CITIES` by the resolved difficulty + region filter.
- **Challenge mode:** the picker is solo-only. In challenge mode difficulty stays fixed to
  the creator's chosen level (same as 2D); the picker is hidden/disabled.

## 4. Play with friends (port 2D challenge mode)

Port the Supabase challenge flow from `georush.html` to the globe, reusing logic verbatim
where possible.

- **Mode detection on load:** read `?c=ID` from the URL. No param → solo; param → challenge.
- **Create:** a Friends/⚙️ UI collects nickname + region + rounds, calls `createChallenge()`
  (`generateChallengeId` + `sbPost`), and shows a shareable `georush3d.html?c=ID` link.
- **Play:** challenge players load the fixed city set via `loadChallenge()` (`sbGet`) and play
  the same cities in order on the globe.
- **Leaderboard:** after the final round, submit the score (`sbPost`) and render the live
  leaderboard (`renderLeaderboard`).
- Reuse `generateChallengeId`, `sbGet`/`sbPost`, `createChallenge`, `loadChallenge`,
  `renderLeaderboard` and the challenge UI markup/state transitions from `georush.html`,
  adapting only the map-layer calls (drop pin / draw arc) to the globe API.
- Same live Supabase URL/anon key and tables (`challenges`, `scores`) — no schema changes.
- All Supabase calls handle null returns gracefully (show a message, never crash the loop).

## Architecture & data flow

- Single standalone `georush3d.html`. Solo state machine and the mobile-fit fix are preserved.
- New challenge states mirror the 2D game: `challenge_create`, `challenge_complete`.
- Difficulty resolution: a `difficultyMode` ('auto' | 'easy' | 'medium' | 'hard'); each round
  resolves the effective level (round-based when 'auto', fixed in challenge mode, else manual).

## Error handling

- Hi-res texture load failure → fall back to low-res texture.
- WebGL/Globe.gl CDN failure → existing `#mapStatus` fallback linking to the 2D version.
- Supabase null returns → user-facing message, game loop never crashes.

## Testing

Extend the Playwright suite (`/tmp/gr3d_test.js`), keeping the real-click and mobile-fit
assertions, and add:
- Zoom floor lets the camera get closer (assert `controls.minDistance` lowered / reachable).
- Auto difficulty follows the round schedule: mock/advance the round counter and assert the
  resolved level matches 1–4 Easy, 5–6 Medium, 7–10 Hard, and that it loops past round 10.
- Manual picker locks the city pool to the chosen level.
- Challenge mode: loading `?c=ID` enters challenge flow; completing rounds renders a
  leaderboard; create-challenge produces a shareable link.
- Run on desktop + iPhone 12 emulation + the CDN-blocked fallback.

## CLAUDE.md updates

Update the `georush3d.html` section: new `minDistance` value, hi-res texture + fallback,
the Manual/Auto difficulty model (round-based schedule), and that challenge mode + Supabase
leaderboards now exist on the globe (no longer solo-only).

## Out of scope / follow-ups

- Alternative map styles (political/vector), re-adding country borders.
- Inlining deps for offline support.
