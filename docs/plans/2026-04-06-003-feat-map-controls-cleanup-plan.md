---
title: "feat: Move map controls onto map and clean up UI"
type: feat
status: active
date: 2026-04-06
---

# feat: Move map controls onto map and clean up UI

## Overview

Move the metric toggle (Moisture/Temperature), tile layer switcher (Street/Satellite), and heatmap toggle out of the card header and Leaflet's default dropdown, and onto the map itself as clearly visible custom controls. Enable heatmap by default. Make the metric toggle also control marker colors (not just heatmap). Keep Canvas/Map toggle in the card header.

## Problem Frame

The current map UI has controls scattered across two locations — the card header (metric toggle) and Leaflet's built-in dropdown (tile layers + heatmap). The dropdown is hidden behind a small icon that users may not discover. The metric toggle above the map feels disconnected from the map content it controls. Consolidating all map-specific controls as visible buttons directly on the map makes the interface self-explanatory.

## Requirements Trace

- R1. Move metric toggle (Moisture/Temperature) onto the map as a custom Leaflet control
- R2. Replace Leaflet's default `L.control.layers` dropdown with explicit visible buttons for Street, Satellite, and Heatmap
- R3. Heatmap is on by default when the map loads
- R4. Metric toggle changes both heatmap colors AND marker colors (currently only heatmap)
- R5. Canvas/Map toggle stays in the card header (no change)
- R6. Remove the `.metric-toggle` and `.legend-row` from the HTML card header (they move onto the map)

## Scope Boundaries

- No canvas fallback logic (future work)
- No new metrics beyond moisture and temperature
- No change to popup content or table behavior

## Context & Research

### Relevant Code and Patterns

- **Current layer control**: `L.control.layers()` in `initLeafletMap()` at `main.js` — will be removed and replaced with a custom control
- **Heatmap toggle**: Currently wired through `layerControl.addOverlay(heatmapOverlay, 'Heatmap')` with `overlayadd`/`overlayremove` events — needs rewiring to a button
- **Metric toggle**: `setupMetricToggle()` in `main.js` reads `.metric-btn` buttons from HTML — will move to a Leaflet custom control
- **Marker colors**: Currently always use moisture via `MOISTURE_COLOR[moistureLevel(...)]` in `updateLeafletMarkers()` — needs to branch on `activeMetric`
- **Heatmap legend**: `HeatmapLegend` custom Leaflet control at bottom-right — already on the map, will stay
- **Custom control pattern**: `HeatmapLegend` extends `L.Control` — follow this same pattern for the new controls

### Layout Design

> *Directional guidance — exact pixel values determined during implementation.*

```
┌──────────────────────────────────────────────────────┐
│ Map                                    ┌────────────┐│
│                                        │ Street     ││
│                                        │ Satellite  ││
│                                        │ ────────── ││
│                                        │ Heatmap ✓  ││
│                                        │ ────────── ││
│                                        │ Moisture ● ││
│                                        │ Temperature││
│                                        └────────────┘│
│                                                      │
│                                                      │
│                              ┌──────────────────┐    │
│                              │ Moisture (%)      │    │
│                              │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │    │
│                              │ 20%          85%  │    │
│                              └──────────────────┘    │
└──────────────────────────────────────────────────────┘
```

- **Top-right**: Single control panel with three sections — tile layer (Street/Satellite), heatmap toggle, and metric selector (Moisture/Temperature)
- **Bottom-right**: Heatmap legend (already exists, no change)
- Style: White background with subtle shadow, rounded corners, matching the existing dashboard card aesthetic

## Key Technical Decisions

- **Single custom `L.Control` for all map buttons**: One control panel in the top-right replaces both `L.control.layers` and the HTML metric toggle. Simpler DOM, one styling target.
- **Heatmap on by default**: Add the overlay layer on init rather than waiting for user click. The `overlayadd` event logic remains but fires on init.
- **Metric toggle controls markers**: `updateLeafletMarkers()` already receives full reading data — branch color logic on `activeMetric` to use temperature color scale when selected.
- **Temperature color scale**: Blue (cold) → Yellow (moderate) → Red (hot). Reuse existing CSS variable colors where possible.

## Open Questions

### Resolved During Planning

- **Where do controls go?** Top-right of the map, as a single grouped panel.
- **Remove old `L.control.layers`?** Yes — replaced entirely by the custom control.
- **Remove metric toggle from HTML header?** Yes — it moves onto the map. The `.legend-row` can also be removed since the heatmap legend on the map serves that purpose.

### Deferred to Implementation

- Exact temperature color thresholds and hex values — derive from the data range in seed nodes
- Whether the control panel needs a collapse/expand toggle on very small screens

## Implementation Units

- [ ] **Unit 1: Create custom map control panel (tile layers + heatmap + metric)**

**Goal:** Replace `L.control.layers` and the HTML metric toggle with a single custom `L.Control` on the map that has buttons for Street, Satellite, Heatmap (toggle), and Moisture/Temperature (radio).

**Requirements:** R1, R2, R3, R6

**Dependencies:** None

**Files:**
- Modify: `static/js/main.js`
- Modify: `static/css/style.css`
- Modify: `templates/index.html`

**Approach:**
- Create a new `MapControlPanel` extending `L.Control` (follow the `HeatmapLegend` pattern). Position: `topright`.
- Panel has three sections separated by thin dividers: (1) tile layer buttons — "Street" and "Satellite" as radio-style buttons, (2) heatmap toggle — "Heatmap" as a checkbox-style button, (3) metric selector — "Moisture" and "Temperature" as radio-style buttons.
- Remove `L.control.layers()` from `initLeafletMap()`. Store tile layer references so the control panel can swap them.
- Remove the `.metric-toggle` div and `.legend-row` from `index.html` since they move onto the map.
- On init, add the heatmap overlay to the map immediately (heatmap on by default). Show the heatmap legend on init.
- Wire the metric buttons to update `activeMetric` and trigger re-render of heatmap + markers.
- Use `L.DomEvent.disableClickPropagation(container)` to prevent map clicks through the panel.

**Patterns to follow:**
- `HeatmapLegend` in `main.js` — extending `L.Control`, creating DOM in `onAdd`, using `L.DomUtil.create`
- `.time-btn` / `.metric-btn` styling — match the existing button aesthetic but at a smaller scale for map overlay

**Test scenarios:**
- Happy path: Map loads with Street tiles active, Heatmap visible, Moisture selected
- Happy path: Clicking "Satellite" swaps tile layer to Esri imagery
- Happy path: Clicking "Heatmap" toggles the overlay off, clicking again restores it
- Happy path: Clicking "Temperature" switches metric and triggers re-render
- Edge case: Rapidly toggling between Street/Satellite doesn't cause flicker or duplicate layers
- Integration: Heatmap legend visibility stays synchronized with the Heatmap toggle button

**Verification:**
- No `L.control.layers` dropdown appears on the map
- No metric toggle or legend row appears in the card header
- All controls are visible on the map without hovering or expanding anything
- Heatmap is visible on initial load

---

- [ ] **Unit 2: Metric toggle controls marker colors**

**Goal:** When the user switches to Temperature, marker colors reflect temperature instead of moisture.

**Requirements:** R4

**Dependencies:** Unit 1 (metric toggle wired on the map)

**Files:**
- Modify: `static/js/main.js`

**Approach:**
- In `updateLeafletMarkers()`, branch on `activeMetric`:
  - `moisture`: existing logic — `MOISTURE_COLOR[moistureLevel(normalizeMoisture(rawMoisture))]`
  - `temperature`: new temperature color function — map temperature to a blue→yellow→red gradient. Define thresholds based on the seed data range (~12°C to ~28°C).
- Add a `temperatureColor(temp)` helper that returns a hex color from a blue-yellow-red scale.
- When `activeMetric` changes (from the control panel), call `updateLeafletMarkers(lastReadings)` to re-color all markers.

**Patterns to follow:**
- `moistureLevel()` + `MOISTURE_COLOR` lookup pattern — create an analogous `temperatureColor()` function
- The heatmap already has temperature color logic in `heatmap.js` — reuse the same color scale for consistency

**Test scenarios:**
- Happy path: With Moisture selected, markers are colored green/yellow/orange/red by moisture level
- Happy path: Switching to Temperature, markers recolor to blue/yellow/red by temperature
- Happy path: Switching back to Moisture restores original marker colors
- Edge case: Node with null temperature — marker falls back to a neutral gray color
- Integration: Marker colors and heatmap colors both update simultaneously when metric changes

**Verification:**
- Markers visually change color when toggling between Moisture and Temperature
- Colors are consistent between markers and heatmap overlay for the same metric

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Custom control panel obscures markers in top-right | Keep the panel compact. Test with all 13 nodes visible. |
| Removing `L.control.layers` breaks heatmap overlay toggle events | The custom panel replaces the event wiring — test that overlay add/remove still works. |

## Sources & References

- Related code: `main.js:initLeafletMap()`, `main.js:HeatmapLegend`, `main.js:setupMetricToggle()`, `heatmap.js`
- Leaflet custom control docs: https://leafletjs.com/reference.html#control
