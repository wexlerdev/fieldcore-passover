---
date: 2026-04-06
topic: map-view
---

# Interactive Map View for Sensor Nodes

## Problem Frame

FieldCore's dashboard currently displays sensor nodes on a 2D canvas with normalized coordinates — a green rectangle with colored circles. While functional, this view provides no geographic context. Users monitoring agricultural field sensors benefit from seeing nodes on a real map where they can understand spatial relationships relative to terrain, roads, and field boundaries. Adding an interactive map view with popups also introduces the first clickable node interaction in the dashboard.

## Requirements

**Map Display**
- R1. Add a toggle (e.g., button or tab) that switches between the existing canvas field view and a new interactive map view
- R2. The map view displays each node as a colored marker at its geographic coordinates
- R3. Marker color reflects the node's current moisture level using the existing color scheme (red/orange/yellow/green)
- R4. The map auto-fits its zoom and center to show all nodes with reasonable padding
- R5. A layer toggle allows switching between street map and satellite tile views

**Marker Interaction**
- R6. Clicking a marker opens a popup showing the node's latest readings: name, moisture %, temperature, battery %, and signal strength. Only one popup is open at a time; clicking a new marker closes any open popup.
- R7. The popup includes a link/button that scrolls to and highlights the corresponding row in the sensor data table

**Schema Cleanup**
- R8. Rename the `x` and `y` columns in the `nodes` table to `latitude` and `longitude` to accurately describe what they store
- R9. Update all backend and frontend references to use the new column names

**Data**
- R10. Node coordinates are set manually (mocked GPS) — no hardware GPS integration required
- R11. The existing seed data coordinates are valid geographic coordinates and should continue to work on the real map

## Success Criteria

- Nodes appear at correct geographic positions on the map
- Users can toggle between the original canvas view and the map view without losing state
- Clicking a marker shows current sensor data and links to the table row
- The schema rename updates all layers (schema, API responses, frontend) so existing features continue to work identically

## Scope Boundaries

- No field boundary polygons or geofencing
- No geocoding or address lookup
- No GPS hardware integration (coordinates are manually set)
- No clustering behavior (node count is small)
- No heatmap or interpolation layers
- The existing canvas view is preserved, not replaced

## Key Decisions

- **Leaflet + OpenStreetMap**: Free, no API key, lightweight, sufficient for the use case. Satellite tiles available via free providers (e.g., Esri). Avoids vendor lock-in and billing complexity of Google Maps or Mapbox.
- **Toggle, not replace**: Keep the existing canvas view as an option. Low cost to maintain both, and the canvas view may be preferred for a quick glance.
- **Rename x/y now**: The project is early enough that a schema change is cheap. Clearer column names prevent confusion as the map feature makes geographic coordinates more prominent.

## Dependencies / Assumptions

- Leaflet JS and CSS can be loaded via CDN (no build step needed, consistent with existing vanilla JS approach)
- Existing seed coordinates (~38.9N, ~92.3W — central Missouri) are realistic for the demo
- The `/api/sensor/latest` endpoint already returns node coordinates alongside readings

## Outstanding Questions

### Deferred to Planning
- [Affects R1][Technical] What is the best toggle UX — a button pair, tab set, or icon toggle? Should match the existing dashboard style.
- [Affects R8][Technical] Should the schema migration be handled as a new `init_db` version or a simple column rename? Verify what migration approach the project uses.
- [Affects R5][Needs research] Which free satellite tile provider works best with Leaflet (Esri, Stamen, etc.)?

## Next Steps

-> `/ce:plan` for structured implementation planning
