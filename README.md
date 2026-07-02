# OVERHEAD // Vector Flight Scope

A single-file, self-contained night-time flight tracker styled like an 80s vector CRT /
mission-control display. A glowing wireframe globe (d3-geo orthographic + world-atlas
coastlines) rendered to `<canvas>` with phosphor-green bloom, amber aircraft glyphs, range
rings, and a compass — meant to be thrown fullscreen on a projector and left running all night.

Everything is in **`index.html`**. No build step, no framework. Only runtime dependencies are
d3 and topojson, both loaded from a CDN.

---

## Run locally (mock data — looks great with no hardware)

`USE_MOCK` defaults to `true`, so it generates believable synthetic traffic out of the box.

```bash
cd fui-flights
python3 -m http.server 8000
# open http://localhost:8000
```

Or any static host — just serve the folder. Double-click the scope to toggle fullscreen.
Click an aircraft to select it.

> An internet connection is needed on first load to pull d3, topojson, the coastline TopoJSON,
> the Share Tech Mono font, and the dark basemap tiles from CDNs. If the coastline file can't
> load, the globe still renders rings + graticule + traffic; if the basemap tiles can't load,
> the scope just stays wireframe-only. Nothing ever crashes the display.

### Dark basemap

A very dark raster basemap (CARTO *dark, no labels* web-mercator tiles — map data
© OpenStreetMap contributors, tiles © CARTO) is stitched offscreen, reprojected
pixel-by-pixel into the scope's orthographic view, dimmed by `BASEMAP_BRIGHTNESS`, and
cached — so per frame it costs a single `drawImage`. It shows the local roads / water /
urban texture around HOME without competing with the phosphor wireframe. Toggle with `BASEMAP`.

---

## Point it at a real tar1090 / readsb receiver

Edit the `CONFIG` block at the top of `index.html`:

```js
const CONFIG = {
  HOME_LAT: 29.95273,     // your house — center of globe + range rings
  HOME_LON: -95.74904,
  USE_MOCK: false,        // <-- turn off mock
  DATA_URL: "http://localhost:8080/tar1090/data/aircraft.json",  // <-- your endpoint
  MAX_RANGE_NM: 60,
  REFRESH_MS: 1000,
  // ...CRT/visual knobs below...
};
```

Set `HOME_LAT` / `HOME_LON` to your receiver's location, `DATA_URL` to your `aircraft.json`,
and `USE_MOCK: false`. The HUD shows `LIVE` when data is flowing, `NO SIGNAL` (blinking amber)
when it isn't — it keeps retrying every `REFRESH_MS` and never blanks the screen.

### Useful CONFIG knobs

| Key | Default | Purpose |
|---|---|---|
| `UI_SCALE` | `2` | global size multiplier for all text, icons, strokes, and HUD — raise for a bigger projector throw, lower toward `1` for dense traffic |
| `BASEMAP` | `true` | very dark raster basemap (local roads/water) under the wireframe |
| `BASEMAP_BRIGHTNESS` | `0.65` | 0–1 RGB multiplier on the basemap — lower = darker |
| `BASEMAP_GAMMA` | `1.6` | >1 lifts dark detail (roads / water edges) without washing the map out |
| `MAX_RANGE_NM` | `60` | outer range-ring radius; the scope zoom follows it. `60` matches typical home ADS-B coverage; raise it if you have a good outdoor antenna |
| `ALERT_NM` | `10` | contacts inside this turn red |
| `DROP_AFTER_S` | `30` | remove a contact this long after its last position |
| `TRAILS` / `TRAIL_SEC` | `true` / `120` | fading breadcrumb history behind every contact — reads well even when IAH traffic gets dense |
| `PREDICT_MIN` | `2.0` | velocity-vector beam, in minutes of travel — drawn only for the **selected** contact |
| `TYPE_SILHOUETTES` | `true` | draw aircraft-type silhouettes (by ADS-B category) vs. a plain chevron |
| `AIRLINE_TAGS` | `true` | draw small colored airline logo chips on contact labels |
| `SCANLINES` / `VIGNETTE` / `FLICKER` | `true` | CRT overlays (subtle by design) |
| `FLICKER_AMOUNT` | `0.04` | brightness wobble; raise for more, `0` to disable |
| `GLOW` / `GLOW_BLUR` | `true` / `8` | phosphor bloom (`shadowBlur`) |

Emergency squawks (7500/7600/7700) always render red regardless of range.

### Aircraft silhouettes & airline chips

Contacts are drawn as top-down **type silhouettes** chosen from the ADS-B emitter category
that `aircraft.json` provides (the only type hint in the feed): light GA, business jet,
narrow-body, 757, heavy/wide-body, military/fast (delta), and helicopter (with a spinning
rotor). All are inline vector paths — no external image assets — stroked as glowing wireframe
to match the scope. Toggle with `TYPE_SILHOUETTES`.

**Airline chips** are small colored squares (the airline's brand color + IATA code) keyed off
the 3-letter ICAO callsign prefix, listed in the `AIRLINES_DB` map near the top of the script.
Add or edit entries there — `UAL: {iata:"UA", color:"#0033A0"}`. These are generated chips, not
trademarked logo files, so the page stays fully self-contained. GA/military/cargo callsigns
with no match simply get no chip. Toggle with `AIRLINE_TAGS`.

---

## CORS gotcha (read this if you see NO SIGNAL on real data)

`aircraft.json` is served by tar1090. If you serve **this page from a different origin** than
the receiver, the browser blocks the cross-origin fetch (CORS) and you'll get `NO SIGNAL` even
though the receiver is up. Two clean fixes:

**(a) Same origin (simplest).** Serve `index.html` from the *same* host:port as tar1090 — e.g.
drop it alongside the tar1090 files, or reverse-proxy both under one host. Then set:

```js
DATA_URL: "/tar1090/data/aircraft.json"   // same-origin relative path
```

**(b) Same-origin proxy.** Point `DATA_URL` at a tiny proxy on the page's own origin that
fetches `aircraft.json` server-side and re-serves it. (Not included here — a few lines of
nginx `proxy_pass`, or any small script, is enough.)

This is also documented in a comment block near the top of `index.html`.
