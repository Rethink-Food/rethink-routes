# Rethink Food Route Generator — Developer Handoff

**Date:** 2026-05-22
**App:** https://rethink-routes.vercel.app (or your Vercel project URL)
**Repo:** https://github.com/Rethink-Food/rethink-routes

---

## What This App Does

A Flask web app deployed on Vercel that turns a weekly member spreadsheet (.xlsx) into optimized delivery route manifests for Rethink Food's NYC meal delivery operations. Logistics team uploads the member list, the app geocodes addresses, assigns members to 14 routes across Mon-Fri, optimizes each route via TSP, and outputs downloadable manifests + interactive maps.

---

## Architecture

```
api/index.py          ← Single Flask app (Vercel serverless function). ALL logic lives here.
rethink_routes.py     ← Route definitions, geocoding, TSP optimization, constants
templates/
  base.html           ← Dark-theme layout shell + shared CSS
  login.html          ← Password auth
  index.html          ← Upload page (sidebar + landing)
  results.html        ← Results: maps, stop lists, downloads, audit panel
geocode_cache.json    ← Persistent address → lat/lon cache (committed to repo)
vercel.json           ← Serverless config (maxDuration: 300s, single function)
requirements.txt      ← flask, folium, geopy, googlemaps, openpyxl (NO pandas)
```

### Key Constraints

- **Vercel serverless**: read-only filesystem, 250MB function size limit, 300s max execution
- **No pandas**: pandas + numpy exceed Vercel's size limit. All Excel parsing uses openpyxl directly.
- **Python 3.9 compatibility**: Avoid `str | None` union syntax and `dict[str, str]` subscripted builtins in variable annotations. Vercel's Python runtime may be 3.9. Use plain `dict`, `list`, etc.
- **In-memory session store**: `_store: dict = {}` keyed by UUID in session cookie. Results are lost on serverless cold starts. Users see a warning banner about this.
- **Geocode cache**: `geocode_cache.json` is committed to the repo and loaded at startup. On Vercel (read-only FS), new geocodes are cached in memory only. To grow the cache permanently: run locally, let it geocode, commit the updated file.

---

## Data Flow

```
Upload .xlsx → parse_excel() → stops[] + flags[]
                                    ↓
Upload assignments .csv (optional) → prev_assignments dict
                                    ↓
Generate → run_generation(stops, flags)
           ├── Route assignment (ZIP matching → load-balance → borough fallback)
           ├── Geocoding (Google Maps parallel or Nominatim sequential)
           ├── TSP optimization (nearest-neighbour + 2-opt per route)
           └── Returns: results[], kitchen_rows[], flags[], double_deliveries[]
                                    ↓
Results page → per-route maps, stop lists, manifests (.xlsx/.csv),
               kitchen packing list, flags tab, audit panel,
               double deliveries tab, save assignments button
```

---

## The 14 Routes

Defined in `rethink_routes.py` → `ROUTES` list. Each entry:
```python
("letter", "display_name", "borough", "day", ["zip1", "zip2", ...])
```

Routes A-N cover Mon-Fri across Bronx, Brooklyn, Manhattan, and Queens. Some ZIPs appear in multiple routes (e.g., 11433 in both Tue_Jamaica and Fri_Jamaica) — these are load-balanced.

### Route Assignment Priority (in order)
1. **Spreadsheet "Route" column** — if the member has a route letter pre-assigned, use it directly
2. **Previous-week assignments CSV** — preserves routes week-to-week (keyed by member_id + delivery_type)
3. **ZIP_OVERRIDES** — hardcoded exceptions (e.g., Ridgewood → Brooklyn)
4. **ZIP match** — looks up the member's ZIP in the ROUTES zip lists
5. **Borough fallback** — if ZIP isn't in any route, `_zip_borough()` detects the borough from the ZIP prefix and assigns to the least-loaded route in that borough
6. **Dropped** — if the ZIP is completely outside NYC (no borough detected), the stop is flagged and excluded

### Borough Detection by ZIP Prefix (`_zip_borough`)
| Prefix | Borough |
|--------|---------|
| 100, 101, 102 | Manhattan |
| 104, 105 | Bronx |
| 112 | Brooklyn |
| 110, 111, 113, 114, 116 | Queens |
| 103 | Brooklyn (nearest for Staten Island) |

**To add a new ZIP permanently:** add it to the appropriate route's zip list in `ROUTES`. The borough fallback is a safety net, not a permanent solution — auto-assigned ZIPs generate a flag telling you to add them.

---

## Spreadsheet Format

The uploaded `.xlsx` must have these columns:

| Column | Required | Notes |
|--------|----------|-------|
| Member ID | Yes | Unique identifier |
| Status | Yes | Only "Active" rows are processed |
| Box Size | Yes | Large / Medium / Small / 4-Day / Four-Day |
| Address Line 1 | Yes | Street address |
| Address Line 2 | No | Apt/unit |
| City / State / Zip | Yes | |
| Phone Number | Yes | |
| Delivery Instructions | Yes | |
| Available Delivery Days | No | e.g., "Monday & Thursday" — used for preferred-day routing |
| Meal Preferences/Allergens | No | Surfaced as flags |
| Twice delivery | No | "Yes"/"No" — creates two stops for the member |
| Route | No | Pre-assigned route letter (A-N) — overrides ZIP assignment |

### Twice-Weekly Members
If "Twice delivery" = "Yes", the parser creates TWO stop entries:
- "First Delivery" — assigned to the preferred first day (or load-balanced)
- "Second Delivery" — assigned to the preferred second day, with 2-day minimum gap from first

### Box Size Normalization (`clean_box()`)
The function in `rethink_routes.py` normalizes messy spreadsheet values:
- Starts with "l" → Large
- Starts with "m" → Medium
- Starts with "s" → Small
- Contains "four" or matches `4.?da` → Four-Day
- Anything else → Unknown

---

## Key Features

### Geocoding
- **Primary**: Google Maps API (parallel, 10 workers, ~10s for any batch). Requires `GOOGLE_MAPS_API_KEY` env var.
- **Fallback**: Nominatim (sequential, ~2s/address, prone to timeouts)
- **Address cleaning**: `_geocode_query()` extracts just the street (before first comma) + ZIP. This avoids doubled location info in addresses like "150-15 Hillside Ave., Jamaica, NY 11432".
- **NYC bounds check**: Results outside 40.4-41.0 lat / -74.3 to -73.6 lng are rejected
- **Cache**: Successful geocodes cached in `geocode_cache.json`. Failed geocodes (None) are NOT cached so they retry next run.

### Route Optimization
Each route runs nearest-neighbour from the Tribeca depot → 2-opt improvement. Depot start and end points are fixed. Distance shown includes depot legs.

### Manifests
Per-route XLSX and CSV downloads with columns: Stop#, Member ID, Name, Address, Phone, Box Size, Delivery Instructions, Allergens, Tape, Delivery (type), Flag.

**Tape color logic** (`tape_color()` in `rethink_routes.py`):
- Vegetarian/vegan allergen → Green
- Small box → Red
- Medium box → Blue
- Large box → Yellow
- Four-Day box → White

### Audit Panel
10 automated checks with pass/warn/fail/info statuses:
1. Routes generated
2. Stop count consistency (route list vs kitchen list)
3. Box sizes all valid (no Unknown)
4. Stop limits (hard: 40, soft: 44 with dense clustering)
5. Distance cap compliance
6. Geocoding success
7. Twice-weekly placements
8. Member flags (cancel/hold/allergen)
9. Duplicate member IDs
10. Allergen coverage

### Route Preservation
- **Save**: "Save Route Assignments (.csv)" button exports Member ID, Delivery type, Route letter
- **Restore**: Upload the CSV as "Previous week's assignments" before generating. Keyed by `(member_id, delivery_type)` tuples for twice-weekly members.
- Assignments can be uploaded before or after the member list (stored in `session["pending_assignments"]` if uploaded first)

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `APP_PASSWORD` | Yes | Login screen password |
| `SECRET_KEY` | Yes | Flask session signing key |
| `GOOGLE_MAPS_API_KEY` | Recommended | Enables parallel geocoding. Without it, Nominatim fallback is slow and unreliable. |

Set in Vercel Project Settings → Environment Variables.

---

## Known Issues and Gotchas

### Vercel Deployment Silent Rollbacks
If the build fails (e.g., a SyntaxError at module import time), Vercel silently serves the LAST SUCCESSFUL deployment. There's no visible error. This burned us for hours — every commit appeared to "push" but nothing changed.

**How to detect**: Add a version marker to `templates/index.html` (currently `v2026-05-22d` at the bottom of the sidebar). If the marker doesn't update after deploy, the build is failing.

**Common causes of silent rollback**:
- Python 3.10+ syntax (`str | None`, `dict[str, str]`) — use plain types
- Adding heavy packages (pandas/numpy exceed 250MB limit)
- Import errors in `api/index.py` (the entire module must import cleanly)

### openpyxl and Google Sheets Exports
Google Sheets-exported XLSX files have stale dimension metadata. `openpyxl` with `read_only=True` trusts this metadata and truncates rows. The current code uses `data_only=True` (default mode, NOT read_only) which loads the full XML and reads all rows.

### Geocode Cache on Vercel
Vercel's filesystem is read-only. New geocodes are cached in memory for that warm instance only. To permanently grow the cache: run locally, upload a real member list, then commit the updated `geocode_cache.json`.

### Stops Without Coordinates
Stops that fail geocoding are included in route manifests and Total Stops counts but excluded from the interactive map. They appear at the end of the stop sequence (after optimized stops). Each generates a flag.

---

## Debug Diagnostics (Temporary)

The `/upload` endpoint currently returns a `debug` object in its JSON response:
```json
{
  "file_bytes": 128861,
  "max_row": 997,
  "max_col": 35,
  "sheet_title": "SCK CSV 11_24",
  "total_data_rows": 996,
  "active_rows": 318,
  "stops_produced": 334,
  "skipped_statuses": {"(empty)": 458, "cancel": 126, ...}
}
```
This is useful for diagnosing upload issues. It can be removed once the app is stable — just delete the `_debug` dict from `parse_excel()` and the `"debug": _debug` from the upload response.

---

## How to Add a New ZIP Code or Route

1. Edit `ROUTES` in `rethink_routes.py`
2. Add the ZIP string to the appropriate route's zip list
3. No other changes needed — the app picks it up on next generation
4. Deploy: `git push origin master` (Vercel auto-deploys)

To add an entirely new route, add a new tuple to `ROUTES`:
```python
("O", "Mon_NewArea", "Borough", "Monday", ["zipcode1", "zipcode2"])
```

---

## Recent Commit History (Most Recent First)

| Commit | What it fixed |
|--------|--------------|
| `25c2f19` | Expand _zip_borough to cover all NYC boroughs + keep ungeocodable stops in manifests |
| `43b23e1` | Add parse diagnostics to upload response |
| `39baa28` | Remove pandas (too large for Vercel), rewrite parser with openpyxl default mode |
| `1e4e81b` | Fix deployment: remove .python-version, buffer file bytes |
| `e22cd0d` | Fix Python 3.10+ syntax (`str \| None`) breaking Vercel deployment |
| `51fe13f` | Switch parse_excel from openpyxl read_only to pandas (later reverted) |
| `13a2efa` | Auto-assign unrecognized ZIPs to nearest borough route |
| `6c7e556` | Fix geocoding: use street+ZIP only, fix doubled display address |
| `c8b8971` | Raise stop limit thresholds to 40 hard / 44 soft |
| `be2b129` | Fix route preservation for twice-weekly members |
| `6c31186` | Add previous-week assignments upload |
| `47fd423` | Add preferred-day routing, save assignments, audit panel, Tape column |
| `00bc2ba` | Fix stop counts, box sizes, double deliveries per logistics team feedback |

---

## Potential Next Steps

- **Clean up debug diagnostics** — remove `_debug` from parse_excel once stable
- **Review the Flags tab** — 179 flags on the current spreadsheet. Many are allergen notes (expected), but check for "auto-assigned" entries and add those ZIPs permanently to ROUTES
- **Geocode cache growth** — run locally with the full member list to cache all addresses, then commit `geocode_cache.json`
- **Four-Day box count** — currently showing 0 on the results page. Check if the spreadsheet value matches `clean_box()` patterns (needs "4" and "day" or "four" in the string)
- **Stop reordering** — the drag-to-reorder feature exists (`/reorder/` endpoint) but verify it persists correctly
- **Route reassignment** — the `/reassign/` endpoint exists for moving stops between routes
