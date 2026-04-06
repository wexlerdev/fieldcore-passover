---
title: "feat: Add interactive Leaflet map view for sensor nodes"
type: feat
status: active
date: 2026-04-06
origin: docs/brainstorms/map-view-requirements.md
---

# feat: Add interactive Leaflet map view for sensor nodes

## Overview

Add an interactive Leaflet map view alongside the existing canvas field map, with a toggle to switch between them. Markers show node positions at real geographic coordinates with color-coded moisture levels. Clicking a marker opens a popup with the latest readings and a link to the corresponding table row. As a prerequisite, rename the `x`/`y` database columns to `latitude`/`longitude` for clarity.

## Problem Frame

The dashboard currently renders sensor nodes on a plain 2D canvas with normalized coordinates. Users see colored circles on a green rectangle but get no geographic context — no terrain, roads, or field boundaries. An interactive map view gives spatial context and introduces the first clickable node interaction in the dashboard. (see origin: `docs/brainstorms/map-view-requirements.md`)

## Requirements Trace

- R1. Toggle between canvas view and map view
- R2. Map displays nodes as colored markers at geographic coordinates
- R3. Marker color reflects moisture level (red/orange/yellow/green)
- R4. Auto-fit zoom and center to show all nodes
- R5. Layer toggle: street map vs. satellite tiles
- R6. Marker popup with latest readings (one popup at a time)
- R7. Popup link scrolls to and highlights the corresponding table row
- R8. Rename `x`/`y` columns to `latitude`/`longitude`
- R9. Update all backend/frontend references for the rename
- R10. Coordinates are manually set (mocked GPS)
- R11. Existing seed coordinates work on the real map

## Scope Boundaries

- No field boundary polygons or geofencing
- No geocoding or address lookup
- No GPS hardware integration
- No clustering or heatmap layers
- The existing canvas view is preserved, not replaced

## Context & Research

### Relevant Code and Patterns

- **CDN pattern**: Chart.js loaded via `<script>` tag in `templates/index.html:9`. Leaflet follows the same approach.
- **Two data paths**: Server-rendered (normalized 0-1 via `app.py:normalize_coordinates()`) and API-fetched (raw lat/lon via `GET /api/sensor/latest`). The map view must use the API path.
- **Canvas rendering**: `main.js:drawNodes()` is the shared renderer. `drawMap()` handles API data, `drawMapFromNodes()` handles server-rendered data.
- **Color constants**: `MOISTURE_COLOR` object in `main.js:4-9` — `low: #E53E3E`, `fair: #ED8936`, `good: #ECC94B`, `optimal: #48BB78`.
- **Time button styling**: `.time-btn` class in `style.css` — the toggle buttons should match this pattern.
- **Database init**: `CREATE TABLE IF NOT EXISTS` in `backend/scripts/init_db.py`. No migration framework. Re-create and re-seed is the expected approach.
- **Table row rendering**: `updateTable()` in `main.js:200-236` builds `<tr>` elements dynamically on refresh.

### Coordinate Mapping (Critical)

The existing seed data stores **latitude in `x`** and **longitude in `y`** — inverted from the typical convention where x=horizontal=longitude. The rename `x -> latitude`, `y -> longitude` is correct for the actual data values:

| Column | Current name | Example value | Correct name |
|--------|-------------|---------------|-------------|
| Latitude | `x` | 38.942 | `latitude` |
| Longitude | `y` | -92.325 | `longitude` |

Leaflet's `L.marker()` takes `[lat, lng]` order, so after rename: `[node.latitude, node.longitude]`.

## Key Technical Decisions

- **Leaflet + OpenStreetMap**: Free, no API key, lightweight CDN. (see origin)
- **Esri World Imagery for satellite tiles**: Free tier, well-maintained, widely used with Leaflet. URL: `https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}`. Requires attribution string.
- **Toggle UX: button pair in map card header**: Matches the existing `.time-btn` pattern used for time range selection. Two buttons ("Canvas" / "Map") next to the "Field Sensor Map" title.
- **Schema re-create, not migrate**: No migration framework exists. Update the SCHEMA string in `init_db.py`, delete `sensors.db`, and re-seed. Acceptable for an early-stage project with only seed data.
- **Map fetches from API, not template data**: The server-rendered template path normalizes coordinates to 0-1 (destroying geographic values). The Leaflet map must call `fetchLatest()` for raw lat/lon. The canvas view continues to use both paths as before.
- **Ship rename and map together**: The rename is a prerequisite for the map (cleaner key names in JS), and the blast radius is manageable in a single PR for a project of this size.

## Open Questions

### Resolved During Planning

- **Toggle UX (from origin)**: Button pair in the `.map-card` header, matching `.time-btn` styling. "Canvas" and "Map" labels.
- **Migration approach (from origin)**: Update `init_db.py` SCHEMA, delete DB, re-seed. No ALTER TABLE needed.
- **Satellite tile provider (from origin)**: Esri World Imagery — free, reliable, no API key.
- **Map data source (from review)**: Map uses `fetchLatest()` API (raw coords), not server-rendered template data (normalized 0-1).

### Deferred to Implementation

- Exact Leaflet popup HTML structure — directional guidance in Unit 3, final markup determined during implementation
- Whether `L.circleMarker` or `L.marker` with colored icon looks better — try `L.circleMarker` first for consistency with the canvas circles

## Implementation Units

- [ ] **Unit 1: Rename x/y to latitude/longitude across the stack**

**Goal:** Update the database schema, all backend code, frontend code, tests, and docs to use `latitude`/`longitude` instead of `x`/`y`.

**Requirements:** R8, R9

**Dependencies:** None

**Files:**
- Modify: `backend/scripts/init_db.py`
- Modify: `backend/scripts/seed_db.py`
- Modify: `backend/models/database.py`
- Modify: `backend/routes/nodes.py`
- Modify: `app.py`
- Modify: `static/js/main.js`
- Modify: `docs/API.md`
- Modify: `docs/DATABASE.md`
- Test: `tests/test_api.py`

**Approach:**
- In `init_db.py`: Change SCHEMA column definitions from `x REAL NOT NULL` / `y REAL NOT NULL` to `latitude REAL NOT NULL` / `longitude REAL NOT NULL`
- In `seed_db.py`: Update INSERT column list and any references
- In `database.py`: Update `create_node()` parameter names and INSERT SQL, update `get_latest_readings()` SELECT to explicitly name `n.latitude, n.longitude` instead of relying on `SELECT *` for the join
- In `nodes.py`: Update `required` list to `["node_id", "name", "latitude", "longitude"]`, update validation and `create_node()` call
- In `app.py`: Update `normalize_coordinates()` to reference `r.get("latitude")` / `r.get("longitude")`. In the `index()` route, keep the server-rendered canvas data using `"x"` and `"y"` keys (these are normalized 0-1 positional values, not geographic coordinates): `"x": r.get("latitude", 0.5)`, `"y": r.get("longitude", 0.5)`
- In `main.js`: Update `normalizeCoords()` to read `r.latitude` / `r.longitude`, update `drawMap()` accordingly. Also update `drawMapFromNodes()` — it reads `node.x`/`node.y` from server-rendered data and does not need renaming since the template still emits `x`/`y` keys for canvas positioning
- In `tests/test_api.py`: Update all test payloads from `"x"/"y"` to `"latitude"/"longitude"`
- In docs: Update API.md and DATABASE.md field names
- After all changes: delete `backend/sensors.db` and re-seed

**Patterns to follow:**
- Existing parameter naming in `database.py:create_node()` — rename parameters alongside SQL
- `get_all_nodes()` uses `SELECT *` and `dict(r)` — column names flow through automatically after schema change

**Test scenarios:**
- Happy path: POST `/api/nodes` with `latitude` and `longitude` fields creates a node and returns them in the response
- Happy path: GET `/api/sensor/latest` returns `latitude` and `longitude` keys (not `x`/`y`)
- Happy path: GET `/api/nodes` returns all nodes with `latitude` and `longitude` keys
- Error path: POST `/api/nodes` with old `x`/`y` fields (missing `latitude`/`longitude`) returns 400
- Error path: POST `/api/nodes` with non-numeric `latitude` returns 400
- Happy path: GET `/` dashboard renders without errors (template receives correctly-keyed data)

**Verification:**
- All existing tests pass with updated field names
- `GET /api/sensor/latest` response contains `latitude`/`longitude` keys
- Dashboard loads at `GET /` without errors
- Canvas map renders identically to before the rename

---

- [ ] **Unit 2: Add Leaflet map view with toggle, markers, and tile layers**

**Goal:** Add the Leaflet library, a map container, toggle buttons, and render color-coded markers at geographic coordinates with auto-fit zoom and a street/satellite layer toggle.

**Requirements:** R1, R2, R3, R4, R5, R10, R11

**Dependencies:** Unit 1 (latitude/longitude column names)

**Files:**
- Modify: `templates/index.html`
- Modify: `static/js/main.js`
- Modify: `static/css/style.css`

**Approach:**
- In `index.html`: Add Leaflet CSS `<link>` in `<head>` and Leaflet JS `<script>` before `main.js` (same CDN pattern as Chart.js). Inside `.map-card`, add toggle buttons ("Canvas" / "Map") after `.card-title`. Add a `<div id="leafletMap">` sibling to `.canvas-wrap`. The canvas wrapper and leaflet container are mutually visible — toggle shows one, hides the other.
- In `style.css`: Style the toggle buttons to match `.time-btn`. Set `#leafletMap` to fill the `.map-card` area with a fixed height matching the canvas.
- In `main.js`:
  - Add `initLeafletMap()`: Create `L.map('leafletMap')` with two base layers — OSM street tiles and Esri satellite tiles — using `L.control.layers()`. Disable auto-zoom (will be set by `fitBounds`).
  - Add `updateLeafletMarkers(readings)`: Track markers by `node_id` in an object/Map. On refresh, update existing markers' style and popup content in place rather than clearing and recreating (this preserves open popups). For new nodes, create `L.circleMarker` using `[r.latitude, r.longitude]`, color from `MOISTURE_COLOR[moistureLevel(normalizeMoisture(r.moisture))]`, radius matching `NODE_R` scaled for the map. Call `map.fitBounds()` only on initial render or when the set of node_ids changes (not on every refresh — this preserves user zoom/pan). Pass `{ maxZoom: 15 }` to `fitBounds()` to prevent zooming to street level when only one node exists.
  - Add toggle handler: Toggle `.canvas-wrap` and `#leafletMap` visibility. When switching to map, call `map.invalidateSize()` (Leaflet requires this when container was hidden). Track active view in a variable.
  - Update `refreshDashboard()`: If the map view is active, also call `updateLeafletMarkers()` with the fetched readings. Only call `updateLeafletMarkers()` on successful fetch — on failure, retain last-known markers (the existing try/catch in `refreshDashboard` handles this).
  - The map always fetches raw coordinates from the API via `fetchLatest()` — it never uses the server-rendered template data.

**Patterns to follow:**
- CDN loading: same approach as Chart.js in `index.html:9`
- Toggle buttons: match `.time-btn` class and active-state pattern from the time range bar
- Color palette: reuse `MOISTURE_COLOR` constants from `main.js:4-9`

**Test scenarios:**
- Happy path: Leaflet map initializes and shows OSM tiles when "Map" toggle is clicked
- Happy path: Three markers appear at correct Missouri coordinates (~38.9N, ~92.3W)
- Happy path: Marker colors match moisture levels (green for optimal, red for low)
- Happy path: Map auto-fits to show all markers with padding
- Happy path: Layer control allows switching between street and satellite tiles
- Happy path: Toggling to "Canvas" hides the map and shows the canvas, and vice versa
- Edge case: Toggling back to "Map" after canvas — map renders correctly (invalidateSize called)
- Edge case: Single node — map centers on it at a reasonable zoom level rather than zooming to max
- Edge case: Auto-refresh while map is active updates marker positions and colors without resetting zoom
- Integration: `refreshDashboard()` updates both the active view (canvas or map) and the data table

**Verification:**
- Map displays with tiles loading at the correct geographic area
- Markers are visible and correctly colored
- Toggle switches views cleanly without visual glitches
- Layer control works for both tile sources
- Existing canvas view still works identically when active

---

- [ ] **Unit 3: Add marker popups with table row linking**

**Goal:** Clicking a map marker opens a popup showing the node's latest sensor readings, with a link that scrolls to and highlights the corresponding row in the data table.

**Requirements:** R6, R7

**Dependencies:** Unit 2 (markers exist on the map)

**Files:**
- Modify: `static/js/main.js`
- Modify: `static/css/style.css`

**Approach:**
- In `updateLeafletMarkers()`: Bind a popup to each `L.circleMarker`. Popup HTML shows: node name (bold), moisture %, temperature, battery %, signal RSSI. Include a "View in table" link/button at the bottom.
- Only one popup open at a time (Leaflet's default `L.popup` behavior — `autoClose: true` is the default).
- In `updateTable()`: Add an `id` attribute to each `<tr>` (e.g., `id="row-NORTH_01"`) so the popup link can target it.
- Add a `scrollToNode(nodeId)` function: Finds the row by ID, calls `element.scrollIntoView({ behavior: 'smooth', block: 'center' })`, adds a CSS highlight class (e.g., `.row-highlight`), removes it after 2 seconds via `setTimeout`.
- The popup link calls `scrollToNode(nodeId)` on click.
- In `style.css`: Add `.row-highlight` with a brief background-color transition (e.g., light yellow fade).

**Patterns to follow:**
- Popup content formatting: match the data presentation style from `updateTable()` (percentage symbols, degree symbols, consistent number formatting)
- Highlight animation: CSS transition on `background-color` for a smooth fade effect

**Test scenarios:**
- Happy path: Clicking a marker opens a popup showing name, moisture %, temperature, battery %, signal RSSI
- Happy path: Clicking a second marker closes the first popup and opens the new one
- Happy path: Clicking "View in table" scrolls the page to the correct table row
- Happy path: The target table row gets a visible highlight that fades after ~2 seconds
- Edge case: Node with no readings yet (newly added) — popup shows meaningful placeholder values (e.g., "No data" rather than "undefined%")
- Edge case: Clicking "View in table" when the table row is already visible — row still highlights without jarring scroll
- Integration: After auto-refresh updates the table HTML, the row IDs are preserved so popup links still work

**Verification:**
- Popup appears with correct, formatted sensor data for the clicked node
- Only one popup visible at a time
- "View in table" link scrolls to the correct row and highlights it
- Highlight fades smoothly after a brief duration

## System-Wide Impact

- **Interaction graph:** The 30-second auto-refresh in `refreshDashboard()` must update whichever view (canvas or map) is currently active. Open popups should persist across refreshes if the marker still exists (update popup content in place).
- **Error propagation:** If `fetchLatest()` fails, the map view should retain its last-known markers rather than clearing them. This matches the existing canvas behavior (no error state shown).
- **API surface parity:** The API response shape changes from `x`/`y` to `latitude`/`longitude`. No external consumers exist (early-stage project), so this is safe.
- **Unchanged invariants:** The canvas view, data table, historical chart, time range controls, and all API endpoints continue to work identically. The map is purely additive except for the column rename.

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| CDN unavailability (offline demo) | Leaflet is ~40KB — could vendor locally later if needed. For now, CDN matches the Chart.js approach. |
| Esri satellite tiles rate-limited or unavailable | Street map (OSM) is the default layer. Satellite is a bonus. Degradation is graceful. |
| Column rename breaks something in an untested code path | The rename touches 10 files — enumerate all of them in Unit 1 and run the full test suite. The test coverage is decent (~25 tests). |

## Sources & References

- **Origin document:** [map-view-requirements.md](docs/brainstorms/map-view-requirements.md)
- Related code: `app.py:normalize_coordinates()`, `main.js:drawNodes()`, `database.py:get_latest_readings()`
- Leaflet docs: https://leafletjs.com/reference.html
- Esri tile service: https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer
