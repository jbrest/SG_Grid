{"title": "Singapore RSO grid reference → lat/long & Google Maps PWA", "participants": ["Adam Brest"], "date": "2026-07-15T00:00:00Z", "consensus": "reached", "doc_type": "spec", "summary": "Installable offline PWA that converts Singapore 8-figure grid references (EPSG:3168) into lat/long, long/lat, and ordered Google Maps waypoint links."}
---

# Singapore RSO grid reference → lat/long & Google Maps PWA

## Overview

This spec covers the **validated conversion core, outputs, and delivery constraints** for a small phone-installable web app. It does **not** specify the UI — the owner will define layout, styling, and interaction directly in Claude Code. Implement the logic and constraints below exactly (they are validated against a live ground-truth point); treat everything visual as the owner’s call.

The app takes one or more Singapore local grid references (8-figure, space-separated), converts each to WGS84, and produces three outputs per the requirements. It runs entirely in the browser, needs no backend, and must work offline once installed.

Do **not** assume the input is NATO MGRS. The source calls these “MGR”, but they are *local* SAF grid references on the RSO Malaya system, not UTM-based MGRS. The projection below is what makes the numbers match; MGRS math would not.

## Requirements

### Input
- Accept one grid reference per line; a single line or a list are both valid.
- Primary format: an **8-figure reference** — four digits, a space, four digits, e.g. `3485 5349`.
- Be tolerant: collapse extra whitespace; also accept eight contiguous digits (`34855349`) by splitting 4/4.
- Optional (nice to have, not required for v1): accept 6-figure (3+3) and 10-figure (5+5) references, detected by digit count per half.
- Any line that isn’t two numbers of equal, valid length is flagged with a clear per-line error (state the line and what was expected). Bad lines are skipped, not fatal.

### Reconstruction (grid reference → full RSO easting/northing)
The leading digit of each axis is dropped by convention because it is constant across Singapore. Restore it, scale by the precision, and offset to the **centre** of the grid square:

```
leadE = 6   // hundred-thousands digit, constant for all Singapore eastings
leadN = 1   // hundred-thousands digit, constant for all Singapore northings

// for a half with `d` digits and integer value `v`:
step = 10 ** (5 - d)                 // 8-figure => d=4 => step=10 m
full = lead * 100000 + v * step + step / 2
```

For the primary 8-figure case this reduces to:
```
E = 600000 + E4 * 10 + 5
N = 100000 + N4 * 10 + 5
```

Sanity invariant the app may assert: every valid Singapore easting is in `[600000, 699999]` and every northing in `[100000, 199999]`.

### Conversion (RSO → WGS84)
Use **proj4js**, bundled locally (see Constraints). Register EPSG:3168 with this exact, validated definition string and convert `[E, N]` → `[lon, lat]`:

```
EPSG:3168  Kertau (RSO) / RSO Malaya (m)
+proj=omerc +no_uoff +lat_0=4 +lonc=102.25 +alpha=323.0257905 +gamma=323.130102361111 +k=0.99984 +x_0=804670.24 +y_0=0 +ellps=evrst69 +towgs84=-11,851,5,0,0,0,0 +units=m +no_defs
```

```js
proj4.defs("EPSG:3168", "+proj=omerc +no_uoff +lat_0=4 +lonc=102.25 +alpha=323.0257905 +gamma=323.130102361111 +k=0.99984 +x_0=804670.24 +y_0=0 +ellps=evrst69 +towgs84=-11,851,5,0,0,0,0 +units=m +no_defs");
const [lon, lat] = proj4("EPSG:3168", "WGS84", [E, N]);
```

Accuracy floor is ~15 m (the Kertau→WGS84 datum shift). This is inherent and fine for field use — don’t try to “improve” it, and don’t display more than ~5 decimal places of lat/long as if it were meaningful.

### Outputs (three, per point / per list)

**1. Google Maps link.**
- Single point → a search link that drops one pin:
  `https://www.google.com/maps/search/?api=1&query=LAT,LON`
- Multiple points → an **ordered** directions link (waypoints in the sequence given):
  `https://www.google.com/maps/dir/?api=1&origin=LAT1,LON1&destination=LATn,LONn&waypoints=LAT2,LON2|LAT3,LON3|...&travelmode=walking`
  - `origin` = first point, `destination` = last point, intermediate points go in `waypoints` in order, separated by `|` (URL-encode as `%7C`).
  - Travel mode is `walking` or `driving` (owner picks the default / whether it’s user-selectable).
- Google’s URL scheme caps intermediate waypoints at ~9. Beyond that, still emit the link and show a plain warning that Google may drop later stops. Per owner’s instruction, overflowing the cap is acceptable.
- Always also emit **individual per-point pin links** (each a `maps/search` URL) as a fallback that never hits the cap.

**2. lat/long text field** — order `lat, lon`, one point per line, copyable.

**3. long/lat text field** — order `lon, lat`, one point per line, copyable.

Both text fields honour a single **format toggle: decimal ↔ DMS**.
- Decimal: signed or hemisphere-suffixed, ≤5 decimal places.
- DMS: `D°MM'SS.sss"H` with hemisphere letter (Singapore is always `N` and `E`). Match the source’s style, e.g. `1°23'16.772"N` — three decimals on seconds.

Note on axis order: the source writes coordinates **longitude first** (`103°… 1°…`). That habit is exactly why both a lat/long and a long/lat field exist. Label each field unambiguously so a lon-first reader doesn’t misread the lat-first field.

### Non-functional
- **Offline after first load** via a service worker (cache-first).
- **No runtime network calls.** Self-host proj4js — do not load it from a CDN.
- **No browser storage APIs required** (in-memory state only).
- High legibility outdoors; large tap targets (owner’s UI, but keep this in mind).

## Architecture

Static, client-side only. Suggested file set (owner may restructure):

```
index.html            app markup + logic (+ proj4js inlined, or as a local file)
proj4.js              proj4js library, self-hosted (do NOT use a CDN)
manifest.webmanifest  PWA metadata (name, icons, display: standalone)
sw.js                 service worker — MUST be its own file (browser rule); cache-first
icon-192.png          home-screen icon
icon-512.png          install / splash icon
```

Data flow: `input text → split lines → parse → reconstruct RSO E/N → proj4 → {lat,lon} → format outputs + build Maps URLs`.

Hosting: any static HTTPS host — GitHub Pages or Cloudflare Pages. **HTTPS is required** for the service worker to register and for the “install to home screen” + offline behaviour to work; a `file://` open will not install.

## Interface (internal functions)

No external/public API. Suggested internal shape:
- `parseGridRef(line) -> { e4, n4, digits } | { error }`
- `reconstruct(v, lead, digits) -> full`  (or a combined `toRSO(e, n, digits)`)
- `toWGS84(E, N) -> { lat, lon }`  (proj4)
- `formatDecimal(lat, lon)` / `formatDMS(lat, lon)`
- `mapsUrl(points, travelMode) -> string`  and  `pinUrl(lat, lon) -> string`

## Data model

A point is transient, in-memory only:
```
{ raw, e4, n4, E, N, lat, lon, error? }
```
No persistence, no accounts, no storage.

## Implementation plan

1. Scaffold the static files, manifest, and service worker; get “add to home screen” working on a phone against an HTTPS host.
2. Bundle proj4js locally; register EPSG:3168 with the string above.
3. Implement parse → reconstruct → convert; wire the invariant assertions.
4. Implement the two formatters (decimal, DMS) and the Maps URL / pin builders.
5. Build the three outputs + copy-to-clipboard; add the waypoint-cap warning and per-point fallback links.
6. Service worker cache-first; verify full offline operation after first load.
7. Deploy to GitHub Pages / Cloudflare Pages.

(UI styling and layout are the owner’s, done in Claude Code.)

## Testing strategy — golden cases

All values below were validated end-to-end in PROJ, proj4js, and against a live surveyed point. Bake them into tests.

**Ground-truth point (Tengah Airbase).** This is the strongest check — a real reference with a known lat/long.
- Input `3485 5349` → reconstruct → RSO `634855 / 153495` → **lat 1.388019, lon 103.708464** (≈ `1°23'16.87"N, 103°42'30.47"E`).
- Independently stated location: `1°23'16.772"N, 103°42'30.449"E`. Assert the computed point is within **10 m** of it (actual offset ≈ 3 m, i.e. rounding within the 10 m grid square).
- Reverse check: `1°23'16.772"N / 103°42'30.449"E` → RSO ≈ `634854 / 153492` → drops back to grid ref `3485 5349`.

**Landmark unit checks (full RSO easting/northing → lat, lon):**

| Location | RSO E / N | lat | lon |
|---|---|---|---|
| City Hall | 650822 / 142994 | 1.29310 | 103.85200 |
| Changi | 666348 / 150872 | 1.36440 | 103.99150 |
| Tuas | 626788 / 146089 | 1.32100 | 103.63600 |
| Woodlands | 643485 / 159017 | 1.43800 | 103.78600 |

**Invariants / parsing:**
- All Singapore eastings ∈ `[600000, 699999]`; northings ∈ `[100000, 199999]`.
- `3485 5349`, `34855349`, and `  3485   5349 ` parse identically.
- Non-numeric or wrong-length halves produce a flagged, non-fatal error.

**URL format:**
- One point → URL starts `https://www.google.com/maps/search/?api=1&query=`.
- Many points → URL starts `https://www.google.com/maps/dir/?api=1&origin=…&destination=…&waypoints=…`, waypoints `%7C`-separated, in input order.

## Open questions (owner’s call)

- **Grid-square anchor:** spec uses the square *centre* (`+step/2`). Switch to the SW corner if that matches how the unit reads references. (At 10 m, a 5 m difference — below the datum floor.)
- **Travel mode:** default `walking` vs `driving`, and whether to expose a toggle.
- **Precision support:** lock to 8-figure, or also accept 6- and 10-figure references.
- **Future input modes:** Brunei grid and raw WGS84 were discussed earlier as later additions; out of scope for v1 but keep the parser layered so a new mode is “add one parser” with the WGS84 stage shared.
- **DMS seconds precision:** 3 decimals proposed (matches the source); confirm.
