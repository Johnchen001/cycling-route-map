# Routing endpoints, verified calls, and the map skeleton

Every command below was run end-to-end and produced output.

## 1. Routing engines

**OSRM / FOSSGIS (preferred).** Free, no key, real global coverage, supports
`overview=full&geometries=geojson` and `steps=true`. The URL path segment is
`driving` even for the bike/foot servers — the profile is fixed server-side.

| mode | base URL |
|---|---|
| bike | `https://routing.openstreetmap.de/routed-bike/route/v1/driving/` |
| foot | `https://routing.openstreetmap.de/routed-foot/route/v1/driving/` |
| car | `https://routing.openstreetmap.de/routed-car/route/v1/driving/` |

Coordinates in the path are `lon,lat;lon,lat;...` (longitude FIRST).

```bash
# one leg, distance only
curl -s -m 60 -A "hermes/1.0" \
  "https://routing.openstreetmap.de/routed-bike/route/v1/driving/121.5006,31.2827;121.51407,31.25182?overview=false"

# full geometry + turn-by-turn road names
curl -s -m 60 -A "hermes/1.0" \
  "https://routing.openstreetmap.de/routed-bike/route/v1/driving/121.5006,31.2827;121.51407,31.25182?overview=full&geometries=geojson&steps=true"
```

Response shape: `routes[0].distance` (metres), `.duration` (seconds),
`.geometry.coordinates` (`[lon, lat]` pairs), `.legs[0].steps[].name` for roads,
and `waypoints[].location` + `.name` telling you where each input point snapped.

**BRouter (fallback)**, supports named cycling profiles:

```bash
curl -s -m 40 "https://brouter.de/brouter?lonlats=121.5006,31.2827|121.51407,31.25182&profile=trekking&alternativeidx=0&format=geojson"
```

`profile=trekking` (bike), `profile=fastbike`, `profile=hiking-beta` for foot.
GeoJSON comes back with `track-length` (metres) and `total-time` (seconds) in
`features[0].properties`.

## 2. Drive both from Python safely

```python
import json, subprocess

def curl(url):
    out = subprocess.run(["curl", "-s", "-m", "60", "-A", "hermes/1.0", url],
                         capture_output=True, text=True).stdout
    return json.loads(out)
```

Use this instead of `urllib.request` — Python's bundled SSL fails the handshake
against these hosts while `curl` succeeds.

## 3. Leg-matrix helper (run this before writing anything)

```python
P = {"A": (lat, lon), "B": (lat, lon), ...}

def km(a, b):
    url = BIKE + f"{P[a][1]},{P[a][0]};{P[b][1]},{P[b][0]}?overview=false"
    return round(curl(url)["routes"][0]["distance"] / 1000.0, 2)
```

Print a table over the candidate pairs, assemble the sequence that hits the
target, then fetch full geometry only for that sequence. Cached leg numbers let
you re-order stops without more network calls.

## 4. Picking the screenshot window size

Ground resolution in Web Mercator: `156543.03 * cos(lat) / 2**z` metres/pixel;
a tile is 256 px. So a route's pixel span at zoom `z` is

```
px_x = (lon_span_deg * 111320 * cos(lat)) / (156543.03 * cos(lat) / 2**z)
     = lon_span_deg * 111320 * 2**z / 156543.03
px_y = lat_span_deg * 111320 * 2**z / 156543.03
```

Pick the largest zoom whose span still fits inside
`window_size - 2*fitBounds_padding`, and set `--window-size` a little larger.
For a route spanning ~0.058 deg lon x ~0.043 deg lat, zoom 15 gives roughly
1360 x 1160 px — a 1400x1150 window with 70 px padding fits it.

A route spanning ~0.134 deg lon x ~0.061 lat only fits at zoom 14 (~1550 x 830
px). Widen the window rather than dropping the route into the corner: 1700x1080
at `--force-device-scale-factor=1.5` produced a 2550x1620 PNG whose street labels
survived chat compression. Match the window aspect to the bbox — a wide urban or
coastal loop wants landscape, a point-to-point N-S route wants portrait.

## 5. Leaflet map skeleton

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<div id="map"></div>
<div id="info" class="panel"><!-- title, total km, 3-4 short bullets. NO leg table here: a
     multi-row table goes illegible the moment chat downscales the PNG. --></div>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
var route = [[lat, lon], ...];          // Leaflet wants [lat, lon]
var map = L.map('map', {zoomControl: false});
L.control.zoom({position: 'topright'}).addTo(map);   // keep clear of the panel
L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png',
  {maxZoom: 19, attribution: '&copy; OpenStreetMap contributors'}).addTo(map);
L.polyline(route, {color: '#ffffff', weight: 11, opacity: .95}).addTo(map);  // casing
L.polyline(route, {color: '#e8341b', weight: 6, opacity: 1}).addTo(map);     // route
stops.forEach(function (s) {                        // numbered divIcon pins
  L.marker([s.lat, s.lon], {icon: L.divIcon({className: '',
      html: '<div class="pin">' + s.n + '</div>',
      iconSize: [26, 26], iconAnchor: [13, 13]})})
    .addTo(map).bindPopup('<b>' + s.title + '</b><br>' + s.sub);
});
map.fitBounds(L.polyline(route).getBounds(), {padding: [70, 70]});
</script>
```

Style the `.pin` as a filled circle with a white border and a drop shadow, and
give the start/end pin a distinct colour. On a loop the first and last waypoint
are the SAME coordinate, so nudge the closing pin (~+0.0008 lat / -0.001 lon) or
one pin completely hides the other and the loop reads as unfinished. Build the HTML from Python with
`json.dumps` for the coordinate list so the numbers are never hand-typed.

## 6. Headless Chrome screenshot

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --hide-scrollbars \
  --force-device-scale-factor=1.6 --window-size=1400,1150 \
  --virtual-time-budget=25000 \
  --screenshot=/tmp/route/map.png "file:///tmp/route/map.html"
```

Confirm the artifact really is an image and note its dimensions before sending:

```bash
python3 -c "import struct;d=open('/tmp/route/map.png','rb').read(33);print(struct.unpack('>II',d[16:24]))"
```

Shrink for chat: `sips -Z 1700 map.png`. Attach the full-size original too if the
user may want to zoom into street names.

## 7. GPX export

```python
pts = "\n".join(f'      <trkpt lat="{p[1]:.6f}" lon="{p[0]:.6f}"></trkpt>' for p in coords)
gpx = f'''<?xml version="1.0" encoding="UTF-8"?>
<gpx version="1.1" creator="Hermes" xmlns="http://www.topografix.com/GPX/1/1">
  <trk><trkseg>
{pts}
  </trkseg></trk>
</gpx>'''
```

`coords` here is the OSRM `[lon, lat]` list — write `lat` from `p[1]` and `lon`
from `p[0]`, the swap is the single easiest thing to get wrong in this whole
procedure.

## 8. Fixing POI coordinates with Overpass

Nominatim is not dependable for Chinese POIs (wrong province, then empty
results). Overpass returns the real OSM centre, and `urllib` is fine here — the
curl rule in §2 applies only to the routing hosts.

```python
import json, urllib.parse, urllib.request

Q = '''[out:json][timeout:60];
(
 nwr["name"~"上海邮政博物馆"](31.23,121.47,31.26,121.50);
 nwr["name"~"大宁灵石公园|大宁公园"](31.25,121.43,31.30,121.48);
);
out center 40;'''
req = urllib.request.Request("https://overpass-api.de/api/interpreter",
                             data=urllib.parse.urlencode({"data": Q}).encode(),
                             headers={"User-Agent": "hermes/1.0"})
for e in json.load(urllib.request.urlopen(req, timeout=90))["elements"]:
    c = e.get("center") or e          # `out center` puts ways/relations' centre here
    print(e["type"], e.get("id"), e.get("tags", {}).get("name"), c["lat"], c["lon"])
```

- Bbox is `south,west,north,east`; keep it city-sized. A name regex over a whole
  country returns dozens of unrelated hits you then have to filter by eye.
- `out center` is what gives ways and relations a usable coordinate; plain nodes
  carry lat/lon themselves.
- Retry once on 504 — `overpass.kumi.systems` and `overpass.osm.jp` are the
  fallback mirrors, but the retry against the primary usually just works.
- Put every POI you need into ONE bbox block instead of one request each.
- Trust the OSM centre over a remembered landmark coordinate: reciting coordinates
  from memory put one stop ~400 m out and another ~1.7 km out, which is enough to
  move a pin across a river.

