---
date: 2026-04-06
topic: heatmap-layer
---

# Interpolated Heatmap Layer

## Problem Frame

FieldCore UI's Leaflet map shows individual sensor markers color-coded by moisture level, but operators cannot see conditions *between* sensors. With only 3 nodes spread across fields, the gaps between markers are where most of the land is. An interpolated heatmap surface answers "what are conditions likely like across the whole field?" — turning point data into spatial intelligence.

## Requirements

**Heatmap Rendering**
- R1. Render an IDW (Inverse Distance Weighting) interpolated surface on the Leaflet map using the latest reading from each node
- R2. The heatmap covers the bounding area of all nodes, with reasonable padding beyond the outermost sensors
- R3. The heatmap updates on each auto-refresh cycle (currently 30s) alongside the existing markers and table

**Metric Selection**
- R4. Toggle buttons labeled "Moisture" and "Temperature" in the map card header area, styled consistently with the existing time-range buttons
- R5. Selecting a metric re-renders the heatmap surface using that metric's values and color scale
- R6. Default metric on page load is moisture

**Color Scales**
- R7. Moisture uses a diverging red → yellow → green gradient (red = dry/low, green = optimal/high), consistent with the existing marker color semantics
- R8. Temperature uses a diverging blue → yellow → red gradient (blue = cold, red = hot)
- R9. Color scales are continuous gradients, not discrete steps

**Legend**
- R10. A color bar legend is displayed on or near the map showing the current metric's gradient with labeled min/max values
- R11. The legend adapts when switching metrics — showing the appropriate color scale, unit label (% or °C), and the actual min/max values from the current data
- R12. The legend updates on each data refresh to reflect current value ranges

**Layer Behavior**
- R13. The heatmap is a toggleable overlay layer, accessible via the existing Leaflet layer control
- R14. Existing circle markers remain visible on top of the heatmap with their current popup behavior unchanged
- R15. The heatmap layer is off by default — the user opts in via the layer control

## Success Criteria

- An operator can glance at the map and immediately understand moisture or temperature conditions across the entire field, not just at sensor locations
- Switching between moisture and temperature is instant and intuitive
- The heatmap does not obscure or break existing map functionality (markers, popups, layer switching, tile layers)
- The visualization looks credible for a capstone demo with 3 nodes

## Scope Boundaries

- **Out:** Historical/time-range heatmaps — the heatmap always shows the latest reading; the chart handles historical trends
- **Out:** Hover-to-inspect interpolated values at arbitrary points (can be added later)
- **Out:** Opacity control / transparency slider (can be added later)
- **Out:** Additional metrics beyond moisture and temperature (battery, signal)
- **Out:** Canvas view changes — the heatmap is Leaflet-only

## Key Decisions

- **IDW interpolation over point-density heatmap:** With only 3 nodes, point-density (Leaflet.heat) produces 3 isolated blobs. IDW creates a continuous surface that's actually useful for understanding field conditions.
- **Toggleable overlay, not marker replacement:** Keeps backward compatibility and lets users see both the detailed markers and the spatial overview simultaneously.
- **Live data only:** Avoids the complexity of interpolating historical averages, which adds API work and questionable user value ("what does a 7-day average heatmap tell a farmer?"). The chart already serves historical analysis.
- **Off by default:** The heatmap is opt-in via the layer control to avoid overwhelming users who prefer the simpler marker view.
- **Diverging color scales:** Red-yellow-green for moisture matches the existing marker semantics. Blue-yellow-red for temperature follows meteorological conventions.

## Dependencies / Assumptions

- The existing `/api/sensor/latest` endpoint returns all data needed (moisture, temperature, latitude, longitude per node)
- IDW interpolation will be computed client-side in JavaScript — the server does not need to change
- At least 3 nodes with valid coordinates are needed for a meaningful interpolated surface

## Outstanding Questions

### Deferred to Planning
- [Affects R1][Needs research] What IDW implementation approach works best in a Leaflet canvas overlay? Options include rendering to an off-screen canvas, using a library like leaflet-idw, or computing a grid and drawing rectangles.
- [Affects R2][Technical] What padding factor beyond the node bounding box produces a natural-looking surface without excessive extrapolation into areas with no sensor coverage?
- [Affects R9][Technical] How many grid cells are needed for the interpolation to look smooth without degrading performance on refresh?

## Next Steps

-> `/ce:plan` for structured implementation planning
