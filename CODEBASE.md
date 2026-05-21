# NB Fire Map — Codebase Reference

This document describes the architecture, module structure, data flows, and configuration for the NB Fire Map application. It is intended for developers maintaining or extending the codebase.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [File Layout](#2-file-layout)
3. [Module Breakdown (app.js)](#3-module-breakdown-appjs)
4. [Data Flow](#4-data-flow)
5. [External Dependencies](#5-external-dependencies)
6. [Configuration Reference](#6-configuration-reference)
7. [CSS Custom Properties](#7-css-custom-properties)
8. [Key Functions by Module](#8-key-functions-by-module)
9. [PDF Export Pipeline](#9-pdf-export-pipeline)
10. [Historical / Archive Mode](#10-historical--archive-mode)
11. [Adding New Fire Data Sources](#11-adding-new-fire-data-sources)

---

## 1. High-Level Architecture

NB Fire Map is a **single-page vanilla JavaScript application** (no build step, no bundler) hosted on GitHub Pages at `www.nbfiremap.ca`.

```
index.html          ← Shell: CDN script/style tags, skeleton HTML
style.css           ← All CSS including Leaflet overrides and CSS custom properties
app.js              ← Entire application logic (~8 400 lines)
```

The page uses a single `DOMContentLoaded` listener in `app.js` that initialises every module in sequence. All state lives in closures inside that listener — there is no global mutable state except a handful of `window.*` exports used for cross-module access.

### Rendering stack

| Concern | Library |
|---|---|
| Map tiles & vector layers | Leaflet.js 1.9 |
| Marker clustering | Leaflet.MarkerCluster |
| Basemap imagery | Esri Leaflet + ArcGIS Tile API |
| "Locate me" control | Leaflet.locatecontrol |
| PDF generation | jsPDF + jsPDF-AutoTable |
| Icons | Font Awesome 6 Free (async CDN) |
| Analytics | Google Analytics 4 (`G-RZY5JKNDQ7`) |

---

## 2. File Layout

```
app.js                   Main application logic
index.html               Page shell with all CDN imports
style.css                Stylesheet
CNAME                    GitHub Pages custom domain (www.nbfiremap.ca)
robots.txt               Crawler permissions
sitemap.xml              Single-URL sitemap for SEO
og-preview.png           Open Graph / Twitter card preview image (703×630)
README.md                User-facing readme

511/                     Cached NB 511 data (refreshed by app at runtime)
  events.json
  ferries.json
  webcams.json
  winterroads.json

erd/                     Latest ERD (Emergency Response Division) exports
  active_fires.geojson   Active fire locations (ArcGIS FeatureServer)
  fire_locations.geojson All fire locations including extinguished
  GNBfireActSum.json     GNB season summary / benchmarks
  out_fires.geojson      Extinguished fires
  sums_table.json        10-year benchmark table

archive/
  manifest.json          Sorted list of all archived date strings (YYYYMMDD)
  cwfis/                 Daily CWFIS 24-hour hotspot GeoJSON files
  erd/                   Daily ERD active/out fire GeoJSON snapshots
```

---

## 3. Module Breakdown (app.js)

`app.js` is organised as a sequence of **IIFE modules** followed by top-level initialisation code, all inside a single `DOMContentLoaded` callback.

### Section 1 — `window.NBFireMapConstants` (CONFIG)

Lines ~1–230. A plain object assigned to `window.NBFireMapConstants` that holds all application-wide configuration. Aliased locally as `const CONFIG = window.NBFireMapConstants`. See [§6 Configuration Reference](#6-configuration-reference).

### Section 2 — `window.NBFireMapUtils` (Utilities)

Lines ~230–330. Pure, side-effect-free helper functions and constants exported to `window.NBFireMapUtils`.

| Function | Description |
|---|---|
| `degToCompass(deg)` | Degrees → 16-point compass rose string |
| `toNum(v, d)` | Number formatter with locale thousands separator |
| `fmtDateTime(ms)` | Timestamp → locale date+time string (no timezone) |
| `fmtDateTimeTz(ms, tz)` | Timestamp → date+time string with explicit timezone |
| `fmtDateTZ(ms, tz)` | Timestamp → date-only string with explicit timezone |
| `ymdInTz(ms, tz)` | `{y, m, d}` decomposition in a given timezone |
| `sameYMD(a, b, tz)` | True if two timestamps fall on the same calendar day |
| `startOfTodayUTCfromTz(tz)` | UTC epoch ms for 00:00 in the given timezone |
| `norm(s)` | Lowercase + trim (used for status key comparisons) |
| `escHTML(s)` | Escape `&`, `<`, `>` for safe HTML insertion |
| `isMobile()` | True when viewport width < 768 px |
| `hexToRgb(hex)` | `'#rrggbb'` → `[r, g, b]` (used by jsPDF which needs numeric RGB) |
| `ATLANTIC_TZ` | `'America/Moncton'` constant |

### Section 3 — `LayerManager`

Lines ~330–820. Manages Leaflet layer groups and the basemap toggle.

Key responsibilities:
- Basemap switching between **Esri Imagery** and **OpenStreetMap**
- Crown land polygon staged/lazy loading (only fetches when the layer is toggled on)
- `fireClusters` — the `L.markerClusterGroup` holding all fire markers
- Cluster icon creation with severity-weighted colour blending

Exports: `LayerManager.setBasemap`, `LayerManager.getCrownLandLayer`, `LayerManager.fireClusters`, etc.

### Section 4 — `DataLoadingManager`

Lines ~820–1800. Handles all non-fire data sources.

| Function | Data source |
|---|---|
| `loadFerries()` | `511/ferries.json` — NB 511 ferry locations |
| `loadEvents()` | `511/events.json` — Road closures and incidents |
| `loadCwfisData()` | CWFIS WFS — VIIRS/MODIS 24-hour hotspots |
| `refreshVisibleCwfis()` | Re-queries CWFIS for the current map bounds |
| `loadWinterRoads()` (internal) | `511/winterroads.json` |
| `getWinterRoadColor(cond)` | Condition string → colour from `WINTER_COLORS` |
| `getEventCategory(type, sub, date)` | Classifies a 511 event for icon colouring |

Also exports `WINTER_COLORS` and `POINT_COLORS` for use in legends.

### Section 5 — `FireDataManager`

Lines ~1800–2550. Owns the canonical fire data store and all fire-related logic.

**State:**
- `fireStore` — `Map<id, FireRecord>` — the single source of truth for all fire data

**FireRecord shape:**
```js
{
  id,          // string — OBJECTID or FIRE_NUMBER_SHORT from ERD
  latlng,      // L.LatLng
  layer,       // Leaflet marker
  statusKey,   // normalised status string (e.g. 'out of control')
  props        // raw properties from the GeoJSON feature
}
```

Key functions:

| Function | Description |
|---|---|
| `processFireGeoJSON(data, overrideStatus, isOut)` | Parses a FeatureCollection, creates markers, populates `fireStore` |
| `createFireMarker(latlng, props, statusKey)` | Creates the `marker-badge` Leaflet DivIcon |
| `bindFirePopup(marker, props, statusKey)` | Attaches a lazily-rendered popup |
| `getStatusColor(statusKey)` | Status string → CSS colour (reads `COLORS`) |
| `getFireCauseStatistics()` | Aggregates cause+area stats across all fires |
| `findERDFireLocation(props)` | Matches an ERD fire to a `fire_locations.geojson` record |
| `cleanFireCause(rawCause)` | Strips French and "Final" suffixes from raw ERD cause field |
| `getFireStore()` | Returns the `fireStore` Map |
| `clearFireStore()` | Empties `fireStore` (used by history mode) |

**COLORS object** (initialised once from CSS custom properties at module start):
```js
const COLORS = {
  oc:   cssVar('--oc'),   // Out of Control  — #ef4444
  mon:  cssVar('--mon'),  // Being Monitored — #9D00FF
  cont: cssVar('--cont'), // Contained       — #FFA500
  uc:   cssVar('--uc'),   // Under Control   — #facc15
  pat:  cssVar('--pat'),  // Being Patrolled — #22c55e
  ext:  '#0000FF'         // Extinguished    — hardcoded blue
};
```

### Section 6 — `UIPanelManager`

Lines ~2552–3370. Manages the two main UI panels:
- **Fire Summary overlay** (`#fireSummaryOverlay`) — statistics, pie charts, fire list
- **Nearby Fires panel** (`#nearbyPanel`) — proximity search results

Uses tracked event listeners (`addTrackedEventListener`) so they can be removed on cleanup.

### Section 7 — `PopupUtils` / `zoomUtils`

Lines ~3370–3490. Helper wrappers for map navigation and popup binding.

`zoomUtils`:
- `flyToTarget(latlng)` — animated fly-to with sensible defaults
- `flyToControlled(latlng, opts)` — fly-to only if current zoom is below a minimum

### Section 8 — Map Initialisation and Controls

Lines ~3490–4750. Initialises the Leaflet map, adds all control overlays, and sets up the legend panel. Defines `map`, `overlays`, `fireClusters`, and the basemap layer groups.

### Section 9 — Feature Layers

Lines ~4750–5570. Adds all vector and raster layers to the map:
- VIIRS/MODIS hotspot circles
- Crown land polygon fill
- Winter road polylines
- Ferry icons and webcam markers
- 511 event markers
- Fire Weather Index (FWI), Fire Behavior Prediction (FBP), Fire Risk raster layers

### Section 10 — Summary, Export, and Help

Lines ~5570–7700. Implements:
- **Map PDF export** — canvas screenshot + jsPDF table (see [§9](#9-pdf-export-pipeline))
- `buildSummaryHTML()` — full fire statistics panel content
- `buildHelpHTML()` — help/glossary panel content
- `buildWeeklyChartSVG()` — SVG bar/line chart for weekly fire trend
- `pieCSSSegments()` — status pie chart segments via conic-gradient CSS
- `fireCausePieSegments()` — cause pie chart segments (uses `CAUSE_COLORS`)
- Excel/CSV export (`collectFireRows`, `collectSummaryStats`)
- Nearby fires search (`cityToFires`)

### Section 11 — Historical Viewer and Season Report

Lines ~7715–8384. The `initHistoricalViewer()` IIFE:
- Loads `archive/manifest.json` to discover available dates
- Replaces live fire data with archived GeoJSON snapshots
- Renders historical fire perimeters and CWFIS hotspots
- `generateSeasonReport()` — aggregates an entire year's archive into a report
- Season report PDF export via jsPDF + AutoTable

---

## 4. Data Flow

### Live fire data load sequence

```
DOMContentLoaded
  └─ loadLocalFires()           (Section 9)
       ├─ fetchLocalAny('active_fires')   → erd/active_fires.geojson
       ├─ fetchLocalAny('out_fires')      → erd/out_fires.geojson
       └─ FireDataManager.processFireGeoJSON()
            ├─ createFireMarker()  → L.DivIcon (marker-badge div)
            ├─ bindFirePopup()     → lazily rendered on first open
            └─ fireStore.set(id, record)
                 └─ fireClusters.addLayer(marker)
```

### Fire status filtering

```
applyFireFilter()
  ├─ Reads checked values from .fire-filter-block checkboxes
  ├─ Iterates fireStore
  │    ├─ Matching status → fireClusters.addLayer(marker)
  │    └─ Non-matching    → fireClusters.removeLayer(marker)
  └─ Triggers refreshSummary() if summary panel is open
```

### Popup render (lazy)

```
user clicks marker
  └─ marker._popup is empty placeholder
       └─ 'popupopen' event
            └─ PopupUtils.buildFirePopup(props, statusKey)
                 ├─ FireDataManager.findERDFireLocation()   ← match fire_locations.geojson
                 ├─ FireDataManager.findGNBFireActivity()   ← match GNBfireActSum.json
                 └─ Renders HTML into popup._content
```

---

## 5. External Dependencies

All loaded via CDN in `index.html` — no local node_modules, no build step.

| Library | Version | CDN | Purpose |
|---|---|---|---|
| Leaflet | 1.9.4 | unpkg | Map tiles, vectors, controls |
| Leaflet.MarkerCluster | 1.5.3 | unpkg | Fire marker clustering |
| Leaflet.Locatecontrol | 0.79 | unpkg | "Locate me" GPS button |
| Esri Leaflet | 3.0.12 | unpkg | Esri tile basemap |
| jsPDF | 2.5.1 | cdnjs | PDF generation |
| jsPDF-AutoTable | 3.8.2 | cdnjs | Tables in PDF |
| Font Awesome 6 Free | 6.x | cdnjs | Icons (flame glyph `\uf06d`) |
| Google Analytics 4 | — | google | Page analytics |

---

## 6. Configuration Reference

All values live in `window.NBFireMapConstants` (aliased as `CONFIG` inside `app.js`).

### `CONFIG.MAP`

| Key | Default | Description |
|---|---|---|
| `CENTER` | `[46.6, -66.5]` | Initial map centre [lat, lng] |
| `ZOOM` | `7` | Initial zoom level |
| `MIN_ZOOM` | `5` | Minimum zoom |
| `MAX_ZOOM` | `18` | Maximum zoom |

### `CONFIG.SERVICES`

| Key | Description |
|---|---|
| `FIRES_ACTIVE` | ArcGIS FeatureServer URL for active fire locations |
| `FIRES_OUT` | ArcGIS FeatureServer URL for extinguished fire locations |
| `FIRE_LOCATIONS` | Local `erd/fire_locations.geojson` path |
| `CWFIS_WFS` | CWFIS WFS endpoint for hotspot layers |
| `CROWN_LAND` | Crown land polygon GeoJSON URL |

### `CONFIG.REFRESH`

| Key | Default | Description |
|---|---|---|
| `FIRE_INTERVAL_MS` | `300_000` | Live fire data auto-refresh (5 min) |
| `CWFIS_INTERVAL_MS` | `600_000` | CWFIS hotspot auto-refresh (10 min) |

### `CONFIG.CLUSTERING`

| Key | Default | Description |
|---|---|---|
| `MAX_RADIUS` | `35` | Cluster radius in pixels |
| `SPIDERFY_ON_MAX_ZOOM` | `true` | Spiderfy markers at max zoom |

### `CONFIG.FIRE_STATUS.STATUS_MAP`

Maps raw API status strings to normalised display labels:

```js
{
  'Active - Out of Control': 'Out of Control',
  'Active - Being Monitored': 'Being Monitored',
  // ...
}
```

---

## 7. CSS Custom Properties

Defined in `style.css` on `:root`. The `FireDataManager.COLORS` object reads these once at module init.

| Variable | Value | Used for |
|---|---|---|
| `--oc` | `#ef4444` | Out of Control fire status |
| `--mon` | `#9D00FF` | Being Monitored fire status |
| `--cont` | `#FFA500` | Contained fire status |
| `--uc` | `#facc15` | Under Control fire status |
| `--pat` | `#22c55e` | Being Patrolled fire status |
| `--border` | `#e5e7eb` | Panel/card borders |
| `--shadow-soft` | box-shadow value | Card shadows |
| `--muted` | `#6b7280` | Secondary text |
| `--modis` | `#ff6600` | VIIRS/MODIS hotspot circles |

> **Note for PDF export:** jsPDF requires numeric RGB values, not CSS variable strings. The `statusColorMap` in the PDF export handler hardcodes the resolved hex equivalents of the above variables. If you change `--oc` etc. in `style.css`, update `statusColorMap` in the PDF export to match.

---

## 8. Key Functions by Module

### Utility (`NBFireMapUtils`)

```
hexToRgb(hex)            '#ef4444' → [239, 68, 68]
norm(s)                  'Out of Control' → 'out of control'
escHTML(s)               Safe HTML string insertion
fmtDateTZ(ms, tz)        Date-only formatted string in Atlantic time
toNum(v, d)              '12345.678' → '12,345.7'
```

### FireDataManager

```
getStatusColor(sk)       norm'd status key → CSS colour string
getFireCauseStatistics() → { causeStats: Map, causeAreas: Map, totalArea }
processFireGeoJSON(fc, overrideStatus, isOut)
                         Parses FeatureCollection, adds markers to fireStore
findERDFireLocation(p)   Cross-references props to fire_locations.geojson record
cleanFireCause(raw)      'Recreation (Anglais / Final)' → 'Recreation'
getFireStore()           Returns the live fireStore Map
```

### Summary / Chart

```
buildSummaryHTML()        async; builds full panel HTML including charts
buildWeeklyChartSVG()     Returns inline SVG string for weekly trend chart
pieCSSSegments(counts)    Returns { css, legendHTML } for status doughnut
fireCausePieSegments(...)  Returns { css, legendHTML, tableHTML } for cause doughnut
CAUSE_COLORS              Module-level constant mapping cause → hex colour
```

### PDF Export

```
STATUS_SEV_ORDER          {'out of control':0, ...} — defined once, used in steps 4+5
statusColorMap            Local hex map for jsPDF colour calls
hexToRgb(hex)             From NBFireMapUtils — converts hex to [r,g,b] for jsPDF
```

---

## 9. PDF Export Pipeline

Triggered by the "Export PDF" button. Runs inside an async handler in Section 10.

```
Step 1 — Await tile rendering
  map.once('moveend') + setTimeout(300 ms) to let tiles paint

Step 2 — Collect enabled fire status checkboxes
  .fire-filter-block input[type="checkbox"][checked]

Step 3 — Collect visible fires
  Iterate fireStore; keep fires inside map.getBounds() matching enabled statuses

Step 4 — Draw canvas
  canvas = L.DomUtil.get('map').__leaflet_events.canvas  (Leaflet internal canvas)
  ctx.scale(2,2)  ← retina
  Draw tile layers → draw vector layers → draw fire badges → draw row-number labels
  canvas.toDataURL('image/jpeg', 0.92) → mapImg
  Falls back to html2canvas on CORS taint

Step 5 — Build jsPDF document
  Page 1 (landscape A4): title, timestamp, map image
  Page 2 (portrait A4): autoTable with fire rows
    Column 0 drawn via didDrawCell: numbered circle in status colour
    STATUS_SEV_ORDER used to sort both canvas (step 4) and table (step 5)
    hexToRgb() converts statusColorMap hex values to jsPDF numeric RGB

Step 6 — Save
  doc.save(`NB_Fire_Map_YYYY-MM-DD.pdf`)
```

### Canvas drawing notes

- `ctx.scale(2,2)` is applied before drawing — all coordinates must account for this
- `map.latLngToContainerPoint(latlng)` returns CSS pixel coordinates that work correctly with the `scale(2,2)` canvas context
- Font Awesome glyph `\uf06d` (fa-fire) requires font weight 900 and the `"Font Awesome 6 Free"` font family; checked at draw time via `document.fonts.check()`
- Falls back to 🔥 emoji if the FA font is not yet loaded

---

## 10. Historical / Archive Mode

The `initHistoricalViewer()` IIFE (Section 11) replaces live data with archived snapshots.

**Archive directory structure:**
```
archive/
  manifest.json                    Sorted array of date strings: ["20250820", ...]
  cwfis/
    24_hour_spots_YYYYMMDD.geojson  CWFIS hotspot snapshot for each date
  erd/
    active_fires_YYYYMMDD.geojson
    out_fires_YYYYMMDD.geojson
    fire_perimeters_YYYYMMDD.geojson
```

**History mode lifecycle:**
1. `enterHistoryMode()` — loads manifest, hides live UI, shows history panel
2. `loadHistoricalDate(dateStr)` — fetches the 4 archive files, calls `processFireGeoJSON`, renders perimeters
3. "Same-day fires" (detected AND extinguished on the archive date) are shown as "Under Control" rather than extinguished so they appear on the map
4. Fire perimeters are spatially filtered to only show perimeters within 5 km of an active fire whose detection time falls within ±2 days of the perimeter's date range
5. `exitHistoryMode()` — clears archive layers, restores live `loadLocalFires()`

**Season report** (`generateSeasonReport()`) iterates all archive dates for a selected year, aggregates daily stats, builds a table + SVG chart, and can export to PDF.

---

## 11. Adding New Fire Data Sources

To add a new ERD or CWFIS-style layer:

1. **Add the URL to `CONFIG.SERVICES`** in the NBFireMapConstants block.

2. **Create a load function in `DataLoadingManager`** following the pattern of `loadFerries()`:
   ```js
   async function loadMyLayer() {
     const data = await fetchWithRetry(CONFIG.SERVICES.MY_LAYER);
     if (!data?.features) return;
     data.features.forEach(f => { /* create marker, add to layer group */ });
   }
   ```

3. **Add a layer group** to the `overlays` object passed to `L.control.layers()` in Section 8.

4. **Register it in the legend** HTML in Section 8 if it needs a custom colour legend.

5. **Add it to the canvas drawing loop** in the PDF export (Section 10, Step 4) if it should appear in PDF exports.

For fire status sources, `FireDataManager.processFireGeoJSON()` already handles the full pipeline — just pass a GeoJSON FeatureCollection and an optional `overrideStatus` string.

---

*Last updated: 2025. Generated from codebase analysis of `app.js` (NB Fire Map, GitHub Pages, `www.nbfiremap.ca`).*
