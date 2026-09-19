---
name: route-map-rendering
description: "Plan a route or render a shareable map image."
version: 1.0.0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [maps, routing, cycling, running, osrm, brouter, leaflet, headless-chrome, screenshot, gpx, wechat]
    category: productivity
    requires_toolsets: [terminal]
    requires_tools: [vision_analyze]
    editorial_name: Route & Map Rendering
    editorial_description: Plan a route to a target distance on the real road network, render it as a shareable map image, and export GPX.
---

# Route & Map Rendering

Produce a route the user can actually ride/walk/drive, at the length they asked
for, delivered as a map image they can look at — not a description of a route.

The deliverable is always **two files plus a text briefing**:
1. a map **PNG** (route drawn on a basemap, numbered stops, distance panel)
2. a **GPX** of the real geometry (importable into any bike computer or app)
3. turn-by-turn-ish text: leg table with distances, what each leg passes,
practical notes

Deliver all of it as ONE message (this user's standing rule), with the files as
absolute `MEDIA:/path` lines.

## Non-negotiables

- **Never state a distance you did not route.** Straight-line sums and eyeballed
  guesses are wrong by 40-100% in dense cities. Compute every leg on the routing
  engine and sum those.
- **Design the route from a leg matrix, not from a wish list.** See step 2.
  Guess-and-check once at the end is how you end up at 31 km when the user asked
  for 20 km.
- **Inspect the rendered PNG before sending.** Load it with `vision_analyze` and
  check: basemap tiles loaded (no grey blocks, no watermark text), the polyline
  is unbroken, all numbered markers present, the info panel readable. Sending a
  broken map is worse than sending none.
- **Verify coordinates by their snap, not by their geocode.** After routing,
  read back `waypoints[].location` and the step road names. If the returned roads
  include a river tunnel or a district you did not intend, the input point
  snapped to the wrong place — fix the coordinate, do not rewrite the narrative.

## Procedure

### 1. Fix waypoints as coordinates, not names

Write the stop list as `(label, lat, lon)`. Do NOT depend on geocoding POI names
in China — see the first pitfall. Use coordinates you are confident about (city
landmarks, known intersections), then let the router snap them to the nearest
routable way.

Sanity-check by routing one leg and printing the step road names before
committing to the whole route.

For the Shanghai rides this user asks for, start from the pre-verified waypoint
bank in `references/shanghai-cycling-waypoints.md` — OSM-sourced coordinates that
have already snapped cleanly, plus the leg costs of two proven loops.

### 2. Build a leg matrix and tune to the target length

Route the candidate legs pairwise and print the km for each. Then assemble a
sequence whose sum lands on the target. Reusable leg costs are gold: once you
know `A->B` and `B->C`, re-ordering to add or drop a stop is arithmetic.

Do this BEFORE writing any prose. Concrete numbers from the matrix decide which
stops survive; a stop that costs 3 km more than budgeted is dropped at this
stage, not after the guide is written.

If the sum still misses the target, add or remove ONE intermediate stop and
re-poll only the affected legs — do not re-route the whole loop.

### 3. Fetch the full geometry of the final sequence

`overview=full&geometries=geojson` for the polyline, `steps=true` for road
names. Concatenate legs (drop the duplicated first point of each leg) into one
coordinate list, and keep the per-leg km/min for the panel and the text.

### 4. Render the map (Leaflet HTML -> headless Chrome PNG)

Build a single self-contained HTML file: Leaflet from CDN, tile layer, a white
casing polyline under a coloured one (legibility on busy basemaps), `divIcon`
numbered pins, an info panel holding ONLY the title / total km / 3-4 one-line
bullets, and `map.fitBounds(polyline.getBounds(), {padding:[70,70]})`.

**Keep the leg table out of the map.** The chat platform downscales the PNG, so a
multi-row table inside the panel turns into unreadable mush — it is the first
thing to become illegible and it spends the map's best real estate on text. The
panel carries title + total + a few bullets at ≥13px; the full leg table goes in
the message body, where the user can read it. On a long route (15+ stops) this is
not a preference, it is the difference between a usable map and a blur.

Then screenshot it — this is the reliable way to get a real image out of HTML:

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --hide-scrollbars \
  --force-device-scale-factor=1.6 --window-size=1400,1150 \
  --virtual-time-budget=25000 \
  --screenshot=/tmp/route/map.png "file:///tmp/route/map.html"
```

`--virtual-time-budget` is what lets tiles finish downloading; without it the
screenshot catches an empty map. `--force-device-scale-factor` is what makes the
label text crisp. Chrome logs harmless `CVDisplayLinkCreateWithCGDisplay failed`
errors on macOS — ignore them and check the artifact instead.

See `references/routing-and-map-render.md` for the verified routing calls, the
HTML skeleton, and the sizing math for picking the window size from the route's
bounding box.

### 5. Verify visually, then package

`vision_analyze` the PNG (see Non-negotiables). Ask about one thing per call, and
when judging panel or label text pass `region=[x1,y1,x2,y2]` in ORIGINAL pixel
coordinates — the whole-image view is downscaled and cannot tell you whether the
panel is readable. Fix tile style / marker overlap / panel overflow and re-render;
re-rendering is cheap, so iterate rather than excuse.

Then:
- Write the GPX (`<trkpt lat= lon=>` per point) so the user can navigate.
- Save both to a stable user-visible directory — `~/Desktop/<主题>/`, distance in
  the filename: `~/Desktop/骑行路线/同济42km骑行环线地图.png` +
  `同济42km骑行环线.gpx`. Not `/tmp`, which is where you iterate.
- Shrink the PNG for chat delivery with `sips -Z 1700` if it is several MB.

### 6. Write the text briefing

Match the shape this user gets elsewhere: a leg table with distances, then one
short paragraph per leg saying what you ride through and what is worth stopping
for, then practical notes. Flag closures and access rules that affect the route
itself (museum closed Mondays; promenades are pedestrian-only).

## Pitfalls

- **Nominatim/OSM geocoding is unreliable for Chinese POI names.** A name query
  can silently return a same-named place in a completely different province (a
  Shanghai landmark resolving to a prison farm elsewhere), and after a handful of
  calls the endpoint starts returning empty. Do not build a route on geocoded
  Chinese names, and do not conclude a POI is unmapped when Nominatim returns
  nothing.
- **Fix Chinese POI coordinates with Overpass — it is the dependable source.**
  Tight bbox + `name` regex + `out center`. Retry the mirror once on a 504: the
  first call after an idle period frequently times out and the retry succeeds.
  Query the short stem with `~`, because the OSM name often differs from the
  colloquial one (a park everyone calls X may be tagged `X灵石公园`). Template:
  `references/routing-and-map-render.md` §8.
- **Call the routing endpoints with `curl`, not Python `urllib`/`requests`.**
  TLS negotiation with `routing.openstreetmap.de` and `brouter.de` fails from
  Python's bundled SSL (`SSLV3_ALERT_HANDSHAKE_FAILURE`); the same URL works via
  `curl`. Shell out to `curl -s -m 60` and `json.loads` the stdout.
- **CartoDB basemaps now need an API key** and render `API KEY REQUIRED`
  watermarks across the map. Use `https://tile.openstreetmap.org/{z}/{x}/{y}.png`
  (with `&copy; OpenStreetMap contributors` attribution) as the default; keep the
  tile URL in one place so a style change is a one-line edit.
- **Move the Leaflet zoom control when a panel owns the top-left**, or it renders
  underneath the info panel. `L.control.zoom({position:'topright'})`.
- **Inner-city return legs are much longer than they look.** A return that reads
  as ~6 km of city riding can route to 11 km once one-way streets and bridge
  crossings are respected. Budget the return leg from the router, and prefer
  routing the return on a different street from the outbound instead of doubling
  back along the riverside road.
- **Check the destination's closed day before recommending it as a stop.**
  Memorials and museums commonly close Mondays and on statutory holidays, and many
  require advance real-name booking — a stop that is shut ruins the plan.
- **Do not force a target length exactly.** Land within a couple of km and state
  the real number; padding with a pointless detour is worse than 19.1 km instead
  of 20.
- **Batch the tuning; do not narrate it.** Route every candidate sequence in ONE
  script run, print all the totals, then choose. A chain of single-leg calls one
  per turn leaves the user watching nothing happen for minutes and they will
  interrupt — put 2-3 whole candidate sequences through the leg helper at once.
- **A loop's start and end pin share a coordinate, so one hides the other.** The
  map then looks like it has no finish. Offset the closing pin by ~0.0008° (or
  give it a distinct `zIndexOffset`) and colour it differently from the start.
- **An implausibly long leg is a finding, not a bug.** When a short hop routes to
  several times its straight-line distance, re-request it with `steps=true` and
  read the road names: the router is usually respecting a real access restriction
  (a ferry, a no-bike road). Surface that restriction in the notes and plan
  around it — do not quietly move the waypoint until the map matches an
  assumption you never checked.
- **Say which roads are actually ridable.** Waterfront promenades, riverside
  boardwalks and pedestrian streets are closed to cycling even when they are the
  scenic line on the map. Name the municipal road to use instead (riverbank roads
  running parallel are usually fine) and state it as a note, not as a detail the
  rider discovers at the barrier — a route that sends someone down a no-bike
  riverside path is not usable.
