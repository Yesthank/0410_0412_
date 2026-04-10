# CLAUDE.md

Guidance for AI assistants (Claude Code) working in this repository.

## Project overview

A single-file, zero-build web app that animates a 3-day Fukuoka trip itinerary (April 10–12) on an interactive Leaflet map. A "moving marker" travels between stops along styled polylines, with per-segment camera movement, day transitions, a progress bar, and play/pause/seek controls. UI copy is mixed Korean/English (the trip notes are in Korean).

There is no backend, build system, package manager, or test suite. The entire application — HTML, CSS, data, and JavaScript — lives in `index.html`.

## Repository layout

```
.
├── index.html    # The entire app (HTML + <style> + data + <script>)
└── CLAUDE.md     # This file
```

That's it. Do not add build tooling, dependencies, or split files unless the user explicitly asks for it.

## Running / previewing

No build step. Options:

- Open `index.html` directly in a browser (works because all deps are CDN-hosted).
- Or serve locally, e.g. `python3 -m http.server` and visit `http://localhost:8000/`.

External dependencies (loaded via CDN, no install needed):
- **Leaflet 1.9.4** — `unpkg.com/leaflet@1.9.4` (CSS + JS)
- **CARTO light_all basemap tiles** — `basemaps.cartocdn.com`

If working offline, both need network access or must be vendored.

## Architecture of `index.html`

The file has four logical sections; keep edits inside the right one.

1. **`<style>` (≈ lines 9–364)** — CSS custom properties on `:root` define the design tokens (colors, fonts, radius, glass effect). Per-day colors live in `--day1`/`--day2`/`--day3`; per-transport colors live in `--flight`/`--subway`/`--walk`/`--bus`. Mobile tweaks live in a single `@media (max-width: 600px)` block at the bottom.

2. **DOM (≈ lines 366–420)** — Fixed overlay panels: `#map`, `.day-panel` (day filter), `.current-day-badge`, `.stop-card` (bottom info card), `.control-bar` (play/seek/speed), `.day-overlay` (full-screen day transition).

3. **Data (≈ lines 430–468), inside an IIFE:**
   - `STOPS` — ordered array of itinerary items. **This is the source of truth for the trip.** Each stop has:
     `{ id, name, nameKr, lat, lng, day (1|2|3), time, icon, transport, description, travelMin }`
     `transport` is one of `'flight' | 'subway' | 'walk' | 'bus' | 'sleep' | null`. `'sleep'` marks end-of-day (triggers day transition, no polyline). `null` is only used on the final stop (trip end).
   - `DAY_INFO` — per-day metadata keyed by day number: `date`, `dateEn`, `dateBadge`, `subtitle`, `color`. When adding a day, you must add an entry here **and** matching `--day{N}` CSS variable plus a button in `.day-panel`.
   - `TRANSPORT_ICONS` / `TRANSPORT_COLORS` — lookup maps used by both route rendering and the moving marker.

4. **Logic (≈ lines 470–1130)** — map init → route construction → markers → the `anim` controller object.

### The `anim` controller

One plain object that owns all animation state and DOM updates. Public-ish methods: `play`, `pause`, `toggle`, `setSpeed`, `nextStop`, `prevStop`, `jumpToStop`, `seekTo`, `filterDay`, `start`. Internal helpers are prefixed with `_`.

Key state: `currentSegment`, `progress` (0–1 within segment), `isPlaying`, `speed`, `dayFilter`, `transitioning`.

Core flow:
- `start()` shows the Day 1 overlay and enters segment 0 but does **not** auto-play.
- `tick(timestamp)` is a `requestAnimationFrame` loop that advances `progress` by `delta * speed / _segmentDuration()`. Duration is derived from `stop.travelMin` with per-transport minimums (see `_segmentDuration`).
- When `progress` reaches 1, `_onSegmentComplete` runs `_dwellAtStop` (1500ms scaled by speed), then advances. If the next stop's `day` differs, it triggers `_showDayTransition` (full-screen overlay, ~2700ms total) instead of just advancing.
- `_updatePosition` → `interpolateRoute(segIdx, t)` walks the precomputed polyline points. `flight` segments use `createBezierArc` to draw an arc and the camera follows the marker; other segments are straight lines and the camera `flyTo`s at segment entry only.
- `_updateRouteVisuals` keeps past/current/future segment opacity in sync.
- `filterDay(day)` shows/hides markers **and** polylines per day; `'all'` shows everything.

### `segments` array

Built once at startup from consecutive `STOPS` pairs. Each entry is `{ points, polyline, transport, fromStop, toStop }`. Sleep/null transport segments are stored as empty placeholders (no polyline) so indices stay aligned with `STOPS[i] → STOPS[i+1]`. **Invariant:** `segments.length === STOPS.length - 1`. `currentSegment` indexes into this array.

## Conventions and gotchas

- **Single IIFE, strict mode.** All JS runs inside one `(function() { 'use strict'; ... })();` — no globals. Keep it that way.
- **No frameworks, no bundler.** Plain DOM (`document.getElementById`, `querySelectorAll`, `addEventListener`). Don't introduce React/Vue/etc.
- **IDs are load-bearing.** Controller methods look up elements by id (`playBtn`, `stopCard`, `cardName`, `badgeDay`, `overlayNum`, `movingIcon`, `progressFill`, etc.). Renaming an id means updating every `getElementById` referencing it.
- **`data-day` and `data-speed` attributes** on buttons are compared as strings in `filterDay` / `setSpeed`. Preserve types (the code uses `String(day)` / `Number(btn.dataset.speed)` explicitly).
- **Korean text.** Descriptions and labels are Korean. Don't translate or "fix" them unless asked. The file is `<meta charset="UTF-8">` and `<html lang="ko">` — keep it.
- **Coordinates are `[lat, lng]`** throughout (Leaflet convention). Don't flip them.
- **Day colors must stay in sync** across three places: the `--day{N}` CSS variable, `DAY_INFO[N].color`, and the `.day-panel button[data-day="N"].active` rule.
- **Transport-to-style mapping** lives in the segment-construction loop (flight=arc, subway=dashed, walk=dotted, bus=solid). To add a transport mode, update `TRANSPORT_ICONS`, `TRANSPORT_COLORS`, the segment-build switch, and `_segmentDuration`.
- **`'sleep'` transport is a control signal**, not a drawable route — it triggers the day transition in `tick()`. Only use it on the last stop of a non-final day.
- **Don't auto-play on load.** `start()` intentionally only shows the Day 1 intro and enters segment 0; playback begins when the user presses play (or arrow/space).
- **Keyboard shortcuts:** Space = toggle, ←/→ = prev/next stop, `1`/`2`/`4` = speed. Preserve these if refactoring event handlers.
- **Idle fade of the control bar** is driven by `_resetIdle()` on `mousemove` / `touchstart`; don't remove those listeners without replacing the wake-up path.

## Common edits — where to look

| Task | Location in `index.html` |
|---|---|
| Add / edit / reorder a stop | `STOPS` array (≈ line 430) |
| Change a day's label/color/subtitle | `DAY_INFO` (≈ line 461) + `--day{N}` CSS var + `.day-panel` button CSS |
| Add a transport type | `TRANSPORT_ICONS`, `TRANSPORT_COLORS`, segment-build `if` chain (≈ line 524), `_segmentDuration` |
| Change animation speed/pacing | `_segmentDuration` (base ms/min, per-transport mins) and `_dwellAtStop` (dwell ms) |
| Change camera behavior | `_enterSegment` (per-segment `flyTo` / `flyToBounds`) and `_updatePosition` (flight follow) |
| Tweak UI chrome (glass, badge, card) | `<style>` section; design tokens on `:root` |
| Mobile layout | `@media (max-width: 600px)` block near end of `<style>` |

## Git workflow

- Default branch: `main`.
- Commits are small and descriptive. When the user asks for a commit, only stage `index.html` (and `CLAUDE.md` when touched) — never `git add -A`.
- Do **not** push or open PRs unless explicitly asked.
