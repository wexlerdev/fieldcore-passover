---
title: "feat: Add IDW interpolated heatmap layer to Leaflet map"
type: feat
status: active
date: 2026-04-06
origin: docs/brainstorms/2026-04-06-heatmap-layer-requirements.md
---

# feat: Add IDW Interpolated Heatmap Layer

## Overview

Add a toggleable IDW (Inverse Distance Weighting) interpolated heatmap overlay to the existing Leaflet map. The heatmap renders a continuous color surface across the field area, letting operators visualize moisture or temperature conditions between sensors — not just at sensor locations. Uses an offscreen canvas rendered to `L.ImageOverlay`, computed client-side with no backend changes.

## Problem Frame

With only 3 sensor nodes spread across fields, the gaps between markers are where most of the land is. The current map shows colored dots at sensor locations but gives no spatial intelligence about conditions in between. An interpolated heatmap surface answers "what are conditions likely like across the whole field?" (See origin: `docs/brainstorms/2026-04-06-heatmap-layer-requirements.md`)

## Requirements Trace

- R1. IDW interpolated surface on Leaflet map using latest readings
- R2. Heatmap covers bounding area with padding beyond outermost sensors
- R3. Updates on each 30s auto-refresh cycle
- R4. Moisture/Temperature toggle buttons in map card header
- R5. Metric switch re-renders heatmap with appropriate color scale
- R6. Default metric is moisture
- R7. Moisture gradient: red → yellow → green (matching existing marker colors)
- R8. Temperature gradient: blue → yellow → red
- R9. Continuous gradients, not discrete steps
- R10. Color bar legend with min/max values
- R11. Legend adapts on metric switch (color scale, unit label, min/max)
- R12. Legend updates on data refresh
- R13. Toggleable overlay via Leaflet layer control
- R14. Existing circle markers remain on top with popups unchanged
- R15. Heatmap layer off by default

## Scope Boundaries

- **Out:** Historical/time-range heatmaps — always shows latest reading
- **Out:** Hover-to-inspect interpolated values
- **Out:** Opacity control / transparency slider
- **Out:** Additional metrics beyond moisture and temperature
- **Out:** Canvas view changes — heatmap is Leaflet-only

## Context & Research

### Relevant Code and Patterns

- `static/js/main.js:333-360` — `initLeafletMap()`: map init, tile layers, layer control
- `static/js/main.js:352-356` — Layer control created with `null` for overlays; needs stored reference + `addOverlay()`
- `static/js/main.js:362-418` — `updateLeafletMarkers()`: marker creation/update pattern
- `static/js/main.js:486-506` — `refreshDashboard()`: 30s refresh cycle, view-gated map updates
- `static/js/main.js:43-52` — `normalizeMoisture()`, `moistureLevel()`: existing normalization
- `static/js/main.js:4-13` — `MOISTURE_COLOR` constants: `#E53E3E`, `#ED8936`, `#ECC94B`, `#48BB78`
- `templates/index.html:29-34` — Map card header with `.view-toggle .time-btn` pattern
- `static/css/style.css:59-84` — `.time-btn` button styles (height 36px, border-radius 8px, `.active` state)
- `static/css/style.css:155-164` — `.map-card-header` flexbox layout

### External References

- Leaflet `L.ImageOverlay` supports `setUrl()` with `canvas.toDataURL()` for dynamic updates
- Leaflet `L.Control.extend()` for custom legend controls
- Leaflet custom panes for z-ordering (`map.createPane()`)
- IDW power parameter p=2 is the GIS standard (ArcGIS default) for environmental data
- leaflet-idw library is abandoned (2016, v0.0.1) — do not use; IDW is ~10 lines to implement

## Key Technical Decisions

- **Offscreen canvas + `L.ImageOverlay`** over `L.GridLayer` or leaflet-idw: Simplest approach for 3 data points. No tile-boundary seams. `setUrl(canvas.toDataURL())` for clean 30s updates. GridLayer is overkill for a single bounded surface. leaflet-idw is abandoned and untested on Leaflet 1.x.
- **Custom pane at z-index 350**: Sits between `tilePane` (200) and `overlayPane` (400) where circleMarkers live. Guarantees heatmap renders below markers without DOM ordering hacks.
- **100x100 grid resolution**: Smooth appearance with 3 data points. IDW computation for 10,000 cells with 3 points is <1ms. No perceptible jank on 30s refresh.
- **IDW power p=2**: Standard for sparse environmental data. p=1 over-averages; p=3 creates artificial Voronoi-like partitions.
- **18% bounding box padding with minimum extent guard**: Natural visual fade-out without excessive extrapolation. Minimum extent prevents degenerate bounds if nodes are collinear.
- **Piecewise linear RGB interpolation**: Multi-stop gradient using the existing `MOISTURE_COLOR` hex values as stops. RGB is simpler than HSL/LCH and avoids perceptual brightness issues with green.
- **Normalize before interpolation**: IDW operates on normalized moisture (0-100%) and raw Celsius for temperature. Legend shows actual data-range min/max, not fixed scale.
- **Legend at `bottomright`**: Top-right occupied by layer control, top-left by zoom. Bottom-right is free.
- **Heatmap opacity hardcoded at 0.55**: Opacity slider is out of scope. 0.55 balances visibility with base map legibility.

## Open Questions

### Resolved During Planning

- **IDW implementation approach**: Offscreen canvas + `L.ImageOverlay` with `setUrl()`. Simplest, no tile seams, integrates with layer control natively.
- **Grid resolution**: 100x100 pixels. Smooth for 3 points, <1ms computation, negligible memory.
- **IDW power parameter**: p=2. GIS standard for sparse environmental data.
- **Padding factor**: 18% of bounding box extent per side. Minimum extent guard of 0.001 degrees (~100m) for collinear nodes.
- **Color interpolation method**: Piecewise linear RGB with multi-stop gradients matching existing color constants.
- **Layer control integration**: Store reference to `L.control.layers()`, call `addOverlay()` dynamically. Change `null` to `{}` for initial overlays parameter.

### Deferred to Implementation

- **Exact heatmap opacity tuning**: Starting at 0.55; adjust based on visual testing against both OSM and satellite basemaps.
- **Legend DOM structure and CSS**: Exact HTML for the gradient bar and labels — implement and iterate visually.
- **Metric toggle button placement**: Requirements say map card header; exact positioning relative to existing Canvas/Map toggle to be resolved during implementation based on visual fit.

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not implementation specification. The implementing agent should treat it as context, not code to reproduce.*

```
Data Flow (on each 30s refresh):
  /api/sensor/latest → [readings array]
       │
       ├──→ updateLeafletMarkers(readings)  [existing]
       │
       └──→ updateHeatmap(readings, activeMetric)  [new]
              │
              ├── extract values: moisture_pct or temperature per node
              ├── compute padded bounding box from node lat/lng
              ├── for each cell in 100x100 grid:
              │     ├── map pixel → lat/lng
              │     ├── IDW interpolate from 3 node values (power=2)
              │     └── normalize to 0..1 → RGB via color stops
              ├── write pixel data to offscreen canvas ImageData
              ├── canvas.toDataURL() → heatmapOverlay.setUrl()
              └── update legend min/max labels

Layer Stack (z-index):
  700  popupPane      ← popups
  600  markerPane     ← (not used, circleMarkers are SVG)
  400  overlayPane    ← circleMarkers (existing)
  350  heatmapPane    ← IDW heatmap ImageOverlay [new custom pane]
  200  tilePane       ← OSM / Satellite tiles
```

## Implementation Units

- [ ] **Unit 1: IDW computation engine and color interpolation**

**Goal:** Create the core IDW interpolation function and piecewise RGB color interpolation — the pure computation layer with no Leaflet dependency.

**Requirements:** R1, R7, R8, R9

**Dependencies:** None

**Files:**
- Create: `static/js/heatmap.js`
- Test: manual browser console verification (no test framework for frontend)

**Approach:**
- `idwInterpolate(x, y, points, power)` — takes normalized grid coordinates and array of `{x, y, value}` points, returns interpolated value
- `valueToColor(t, stops)` — takes 0..1 normalized value and array of `{pos, r, g, b}` stops, returns `{r, g, b}`
- Define `MOISTURE_STOPS` using existing hex colors: `#E53E3E` (0.0), `#ED8936` (0.35), `#ECC94B` (0.55), `#48BB78` (1.0)
- Define `TEMPERATURE_STOPS`: `#4299E1` (0.0), `#ECC94B` (0.5), `#E53E3E` (1.0)
- `renderHeatmapCanvas(readings, metric, gridSize)` — takes readings array, metric name, grid size; returns `{dataUrl, min, max, bounds}`. Internally: computes padded bounds, creates offscreen canvas, iterates grid, calls IDW + color for each cell, writes ImageData, returns toDataURL()
- Normalize moisture via existing formula (raw/700*100) before IDW. Temperature uses raw Celsius.
- Padding: 18% of bounding box extent per side, minimum 0.001 degrees

**Patterns to follow:**
- `normalizeMoisture()` at `static/js/main.js:43-45` for moisture normalization formula
- `MOISTURE_COLOR` at `static/js/main.js:4-9` for exact color hex values

**Test scenarios:**
- Happy path: `idwInterpolate` with 3 points returns value between min and max of inputs
- Happy path: `idwInterpolate` at exact point location returns that point's value
- Happy path: `valueToColor(0, MOISTURE_STOPS)` returns red RGB, `valueToColor(1, ...)` returns green RGB
- Happy path: `valueToColor(0.5, TEMPERATURE_STOPS)` returns yellow RGB
- Edge case: `idwInterpolate` with all points having same value returns that value
- Edge case: `renderHeatmapCanvas` with collinear nodes (lat spread near zero) still produces valid canvas (minimum extent guard)
- Edge case: readings with moisture=0 and moisture=700 produce full gradient range
- Edge case: fewer than 3 readings returns null (no render)

**Verification:**
- Functions can be called from browser console with test data and return expected values
- `renderHeatmapCanvas` returns a valid data URL string starting with `data:image/png`

---

- [ ] **Unit 2: Leaflet heatmap layer integration**

**Goal:** Wire the heatmap canvas into Leaflet as a toggleable `ImageOverlay` in a custom pane, integrated with the layer control.

**Requirements:** R1, R2, R13, R14, R15

**Dependencies:** Unit 1

**Files:**
- Modify: `static/js/main.js` (map init, layer control, refresh cycle)
- Modify: `templates/index.html` (script tag for heatmap.js)

**Approach:**
- In `initLeafletMap()`: create custom pane `heatmapPane` at z-index 350 with `pointerEvents: 'none'`
- Store layer control reference in module-level variable `layerControl`; change overlays param from `null` to `{}`
- Create `L.imageOverlay` with initial empty/transparent data URL, add to `heatmapPane`
- Call `layerControl.addOverlay(heatmapOverlay, 'Heatmap')` — starts unchecked (off by default per R15)
- In `refreshDashboard()`: after marker update, call `updateHeatmap(readings)` which invokes `renderHeatmapCanvas()` and calls `heatmapOverlay.setUrl(result.dataUrl)` + `heatmapOverlay.setBounds(result.bounds)`. Store readings in module-level `lastReadings` variable for metric-switch re-renders
- Also wire `updateHeatmap()` into the `setupViewToggle()` first-map-init path at main.js:472 — this calls `fetchLatest().then(readings => updateLeafletMarkers(readings))` directly, bypassing `refreshDashboard()`. Add `updateHeatmap(readings)` to this `.then()` chain so the heatmap renders on first map view activation
- Only update heatmap when `activeView === 'map'` (matching existing marker refresh pattern)
- Track `activeMetric` state variable (default: `'moisture'`)
- Add `<script src="/static/js/heatmap.js"></script>` before main.js in index.html

**Patterns to follow:**
- `updateLeafletMarkers()` at `static/js/main.js:362-418` for the marker update pattern
- Layer control creation at `static/js/main.js:352-356`
- Pane creation: `map.createPane('heatmapPane'); map.getPane('heatmapPane').style.zIndex = 350;`

**Test scenarios:**
- Happy path: enabling heatmap in layer control renders a colored overlay on the map
- Happy path: disabling heatmap removes the overlay, markers and tiles remain
- Happy path: 30s refresh updates the heatmap surface without flicker
- Edge case: switching from canvas view to map view with heatmap enabled renders correctly after `invalidateSize()`
- Edge case: heatmap with fewer than 3 valid nodes does not render (layer appears empty)
- Integration: heatmap overlay renders below circleMarkers — markers are clickable on top of heatmap, popups open normally

**Verification:**
- Heatmap appears as a checkbox in the layer control labeled "Heatmap"
- Toggling the checkbox shows/hides the interpolated surface
- Circle markers and popups function normally on top of the heatmap
- Refreshing data updates the heatmap surface

---

- [ ] **Unit 3: Metric toggle buttons (Moisture / Temperature)**

**Goal:** Add toggle buttons to switch the heatmap between moisture and temperature visualization.

**Requirements:** R4, R5, R6

**Dependencies:** Unit 2

**Files:**
- Modify: `templates/index.html` (add buttons to map card header)
- Modify: `static/js/main.js` (button click handlers, activeMetric state)
- Modify: `static/css/style.css` (button group styling if needed)

**Approach:**
- Add two buttons ("Moisture", "Temperature") in the `.map-card-header` area, wrapped in a `.metric-toggle` container. Use a distinct `.metric-btn` class (not `.time-btn`) to avoid selector collision with `setupTimeButtons()` at main.js:531 and `setupViewToggle()` at main.js:452, which both use unscoped `.time-btn` selectors. Copy the `.time-btn` / `.time-btn.active` styles to `.metric-btn` / `.metric-btn.active` in style.css
- "Moisture" starts with `.active` class (R6 default)
- Click handler: set `activeMetric`, toggle `.active` class, call `updateHeatmap(lastReadings, activeMetric)` to re-render immediately
- If heatmap layer is not currently enabled, still update `activeMetric` so next enable shows the right metric
- Buttons are always visible in the header when map view is active (consistent with existing toggle pattern)

**Patterns to follow:**
- `.view-toggle .time-btn` at `templates/index.html:31-34` for button HTML structure (use as visual reference only — use `.metric-btn` class to avoid selector collision)
- `setupTimeButtons()` at `static/js/main.js:531-553` for click handler pattern (but scope metric handler to `.metric-btn` only)
- `.time-btn` / `.time-btn.active` at `static/css/style.css:59-84` for styling to duplicate as `.metric-btn` / `.metric-btn.active`

**Test scenarios:**
- Happy path: clicking "Temperature" re-renders heatmap in blue-yellow-red gradient
- Happy path: clicking "Moisture" re-renders heatmap in red-yellow-green gradient
- Happy path: "Moisture" button has `.active` class on page load
- Edge case: switching metric when heatmap layer is off updates internal state without error; enabling layer shows correct metric
- Edge case: switching metric during a data refresh does not cause render glitch

**Verification:**
- Two styled buttons appear in map card header area
- Clicking switches the heatmap color scheme immediately
- Active button state visually indicates selected metric

---

- [ ] **Unit 4: Adaptive color bar legend**

**Goal:** Add a Leaflet control showing a gradient color bar with min/max labels that adapts to the active metric and current data range.

**Requirements:** R10, R11, R12

**Dependencies:** Unit 2, Unit 3

**Files:**
- Modify: `static/js/main.js` (legend control creation and update)
- Modify: `static/css/style.css` (legend styling)

**Approach:**
- Create `HeatmapLegend` extending `L.Control` at position `'bottomright'`
- `onAdd()`: create container div with gradient bar (CSS `linear-gradient` or canvas), min/max labels, and metric unit label
- Expose `update(metric, min, max)` method that changes the gradient colors, labels, and unit (% or °C)
- Call `legend.update()` after every `renderHeatmapCanvas()` with the returned min/max values
- Show legend only when heatmap layer is enabled; hide when disabled. Listen to `overlayadd`/`overlayremove` map events to toggle visibility
- Gradient bar rendered as a div with CSS `linear-gradient` using the same color stops as the heatmap

**Patterns to follow:**
- Existing `.legend-row` styling at `static/css/style.css:131-153` for design language
- `L.Control.extend()` pattern from Leaflet docs

**Test scenarios:**
- Happy path: legend appears at bottom-right when heatmap layer is enabled
- Happy path: legend shows "Moisture (%)" with red-to-green gradient and current data min/max
- Happy path: switching to temperature updates legend to "Temperature (°C)" with blue-to-red gradient
- Happy path: data refresh updates min/max labels to reflect new values
- Edge case: legend hides when heatmap layer is toggled off
- Edge case: legend re-appears with correct state when heatmap is re-enabled

**Verification:**
- Color bar displays correct gradient for active metric
- Min/max labels show actual values from current sensor data, not fixed ranges
- Legend visibility tracks heatmap layer toggle state

## System-Wide Impact

- **Interaction graph:** `refreshDashboard()` gains a new call to `updateHeatmap()` alongside existing `updateLeafletMarkers()` and `loadHistory()`. The new code path is additive — no existing calls are modified.
- **Error propagation:** If `renderHeatmapCanvas()` fails (e.g., <3 nodes), the heatmap simply doesn't render. No error propagates to markers, table, or chart. The overlay remains empty/transparent.
- **State lifecycle:** New state variables: `activeMetric` (string), `lastReadings` (array, updated on each refresh), `heatmapOverlay` (L.ImageOverlay), `heatmapLegend` (L.Control), `layerControl` (L.control.layers ref). All initialized during `initLeafletMap()` except `lastReadings` which is set on first data fetch.
- **Unchanged invariants:** Existing circle markers, popups, tile layers, chart, table, time-range buttons, canvas view, and auto-refresh cycle are not modified. The heatmap is purely additive.
- **API surface parity:** No API changes. `/api/sensor/latest` already returns all needed data.

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Heatmap opacity obscures base map or is too faint | Start at 0.55, visually test against both OSM and satellite. Easy to adjust — single constant. |
| IDW extrapolation beyond sensor coverage looks misleading | 18% padding limits extrapolation. IDW naturally converges to nearest sensor beyond convex hull. |
| New `heatmap.js` file loaded before Leaflet — functions depend on `L` global | Load `heatmap.js` after Leaflet CDN script in `index.html`. Functions that reference `L` are only called from `main.js` after map init. |
| Layer control reference not stored — `addOverlay()` unavailable | Refactor `initLeafletMap()` to store control in module-level variable. Low risk, single line change. |

## Sources & References

- **Origin document:** [docs/brainstorms/2026-04-06-heatmap-layer-requirements.md](docs/brainstorms/2026-04-06-heatmap-layer-requirements.md)
- Leaflet `L.ImageOverlay` API: `setUrl()`, `setBounds()`, `setOpacity()`
- Leaflet `L.Control.extend()` for custom controls
- Leaflet custom panes: `map.createPane()` with z-index control
- IDW standard: ArcGIS Pro documentation on power parameter (p=2 default)
- Color interpolation: piecewise linear RGB with explicit stops
