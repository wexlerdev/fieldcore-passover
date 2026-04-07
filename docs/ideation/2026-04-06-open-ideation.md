---
date: 2026-04-06
topic: open
focus: ~
---

# Ideation: FieldCore UI Improvements

## Codebase Context

Flask-based agricultural sensor dashboard (Python/Jinja2/vanilla JS). SQLite backend with nodes and readings tables. Leaflet map with circle markers, Chart.js history, auto-refresh polling. 3 sensor nodes in Missouri with moisture, temperature, battery, and signal data. Recent migration from x/y to lat/lng coordinates. Business logic (normalization, classification) duplicated between Python and JS. No .env config, no migration framework, monolithic frontend.

## Ranked Ideas

### 1. Kill the Canvas View, Unify on Leaflet
**Description:** Remove the hand-rolled canvas map renderer entirely and make Leaflet the only map. Eliminates ~150 lines across Python and JS, removes normalize_coordinates() which destructively overwrites real lat/lon, kills the view toggle, and removes a redundant API fetch.
**Rationale:** Canvas is strictly less capable than Leaflet. Removing it fixes the lat/lon mutation bug and simplifies the entire map codepath.
**Downsides:** Loses a simpler visual for users who don't need geographic context.
**Confidence:** 90%
**Complexity:** Low
**Status:** Unexplored

### 2. Serve Normalized Data from the API
**Description:** Have /api/sensor/latest return pre-computed moisture_pct, moisture_level, and battery_status alongside raw values. Eliminates the triple-normalization divergence where Python uses configurable min/max, JS hardcodes 700, and the template hardcodes color thresholds.
**Rationale:** Single source of truth for the project's core business concept. Every consumer gets correct, consistent data.
**Downsides:** Couples display logic to backend. Slightly larger API response.
**Confidence:** 92%
**Complexity:** Low
**Status:** Unexplored

### 3. Alert Thresholds and Notifications
**Description:** Configurable alert thresholds that trigger visual/audio notifications when ingest_reading() processes a value crossing a boundary. Dashboard-level alerts (flashing row, banner, sound) with architecture supporting later webhook extension.
**Rationale:** Transforms the dashboard from passive display to active monitoring tool.
**Downsides:** Needs schema additions. Notification UX requires care to avoid alarm fatigue.
**Confidence:** 78%
**Complexity:** Medium
**Status:** Unexplored

### 4. Node Staleness Detection
**Description:** Compute a stale flag per node when latest reading timestamp exceeds 2x expected interval. The CSS class .status-dot.stale already exists but is never used.
**Rationale:** Silent sensor death is the worst failure mode. Low cost, high operational value.
**Downsides:** Requires defining expected reporting interval.
**Confidence:** 88%
**Complexity:** Low
**Status:** Unexplored

### 5. Node Detail / Drill-Down View
**Description:** Click a node to see its individual history and status. Backend already supports per-node history via fetchHistory(range, nodeId) but the UI never exposes it.
**Rationale:** Unlocks existing backend capability with pure frontend work.
**Downsides:** Needs new UI panel. Chart must handle single-node vs all-nodes modes.
**Confidence:** 85%
**Complexity:** Medium
**Status:** Unexplored

### 6. Real-Time Updates via SSE
**Description:** Replace 30s polling with Server-Sent Events. Flask supports SSE natively. Push updates when ingest_reading() writes new data.
**Rationale:** Eliminates latency and wasteful HTTP requests. Makes dashboard feel genuinely live.
**Downsides:** SSE connections consume server resources per client.
**Confidence:** 72%
**Complexity:** Medium
**Status:** Unexplored

### 7. Mobile-Responsive Layout
**Description:** Add media queries and responsive breakpoints. Currently zero @media queries in 390 lines of CSS, fixed two-column flexbox with overflow: hidden.
**Rationale:** Agricultural operators are in the field on phones, not at desktops.
**Downsides:** Requires rethinking two-column layout for small screens.
**Confidence:** 82%
**Complexity:** Medium
**Status:** Unexplored

## Rejection Summary

| # | Idea | Reason Rejected |
|---|------|-----------------|
| 1 | Time-range button CSS class conflict | Bug fix, not an improvement idea |
| 2 | Coordinate normalization destroys lat/lon | Subsumed by "Kill Canvas View" |
| 3 | Auto-seed on empty DB | Minor DX convenience, low leverage |
| 4 | Decouple seed script from raw SQL | Code hygiene, not product improvement |
| 5 | Single config endpoint | Subsumed by "Serve Normalized Data" |
| 6 | Test DB seed fixture | Test infrastructure, not user-facing |
| 7 | PWA / offline capability | Too expensive relative to value |
| 8 | Anomaly detection | ML-adjacent complexity; brainstorm variant |
| 9 | Static frontend / decouple from Flask | Architectural rewrite too expensive |
| 10 | Scale beyond 3 nodes | Too vague as standalone idea |
| 11 | Field zones concept | High complexity; brainstorm variant |
| 12 | Per-node calibration | High complexity; brainstorm variant |
| 13 | Date picker instead of fixed ranges | Lower leverage than survivors |
| 14 | Split main.js into modules | Code organization, not user-facing |
| 15 | Frontend tests | Test infrastructure, not product improvement |
| 16 | .env with python-dotenv | Too small-scope for ideation |
| 17 | Sensor health / RSSI dashboard | Duplicates "Staleness Detection" |
| 18 | Extract service layer | Means-to-an-end, not the end |
| 19 | Database migration framework | Infrastructure, not user-facing |
| 20 | No error state UI | Narrow scope, part of broader UX |
| 21 | Export / download data | Lower leverage than top 7 |

## Session Log
- 2026-04-06: Initial ideation — 40 candidates generated across 4 frames, 28 after dedupe, 7 survived filtering. User selected heatmap layer concept (related to but distinct from survivors) for brainstorming.
