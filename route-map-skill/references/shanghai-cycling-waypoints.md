# Shanghai ride waypoints and ridability (OSM-verified)

Building blocks for rides starting at 同济大学四平路校区. Coordinates came from
Overpass / OSM centres and every one of them snapped cleanly to the bike network.
Re-verify only if a stop looks off on the rendered map.

## Waypoint bank `(lat, lon)`

| stop | lat | lon |
|---|---|---|
| 同济大学四平路校区正门（四平路1239号） | 31.28270 | 121.50060 |
| 杨浦滨江·宁国路（绿之丘一带） | 31.25800 | 121.52500 |
| 秦皇岛路码头（上海船厂/毛麻仓库） | 31.251863 | 121.514067 |
| 北外滩·东大名路（国客中心） | 31.24948 | 121.49295 |
| 外白渡桥 | 31.245307 | 121.485744 |
| 上海邮政博物馆（四川路桥） | 31.246407 | 121.480755 |
| 苏河湾万象天地（慎余里/天后宫） | 31.244370 | 121.473874 |
| 静安大悦城（西藏北路166号） | 31.245703 | 121.467573 |
| 四行仓库抗战纪念馆（光复路21号） | 31.242264 | 121.466925 |
| 天安千树（莫干山路·昌化路桥） | 31.250870 | 121.440886 |
| M50创意园（莫干山路50号） | 31.249880 | 121.444743 |
| 长寿路桥 | 31.24150 | 121.43000 |
| 中山公园 | 31.223319 | 121.415624 |
| 华政·苏州河湾（华东政法长宁校区） | 31.227776 | 121.412498 |
| 半马苏河公园 | 31.221706 | 121.393934 |
| 长风公园（大渡河路） | 31.227186 | 121.391360 |
| 大宁灵石公园（广中西路） | 31.276995 | 121.439298 |

Note 大宁灵石公园: the colloquial name is 大宁公园, OSM tags the relation
`大宁灵石公园` — query `大宁灵石公园|大宁公园` or you get nothing.

## Leg costs (bike profile, km)

| from → to | km |
|---|---|
| 同济正门 → 宁国路滨江 | 4.5 |
| 宁国路滨江 → 秦皇岛路码头 | 2.0 |
| 秦皇岛路码头 → 北外滩 | 2.3 |
| 北外滩 → 外白渡桥 | 1.0 |
| 外白渡桥 → 邮政博物馆 | 0.6 |
| 邮政博物馆 → 万象天地 | 0.8 |
| 万象天地 → 静安大悦城 | 0.8 |
| 大悦城 → 四行仓库 | 0.5 |
| 四行仓库 → 天安千树 | 3.6 |
| 天安千树 → M50 | 0.4 |
| M50 → 长寿路桥 | 2.2 |
| 长寿路桥 → 中山公园 | 3.0 |
| 中山公园 → 华政河湾 | 0.7 |
| 华政河湾 → 半马苏河公园 | 2.8 |
| 半马苏河公园 → 长风公园 | 0.7 |
| 长风公园 → 大宁灵石公园 | 9.3 |
| 大宁灵石公园 → 同济正门 | 7.6 |
| 四行仓库 → 同济正门（内陆：西藏北路/海宁路/溧阳路/四平路） | 7.0 |
| 秦皇岛路码头 → 同济正门（最短） | 5.1 |

Arithmetic for two common targets: the riverside loop through 宁国路 → 秦皇岛路
→ 北外滩 → 外白渡桥 → 万象天地 → 四行仓库 sums to ~19.1 km; adding the western
苏州河 stretch (天安千树 → M50 → 长寿路 → 中山公园 → 华政 → 长风 → 大宁) brings
it to ~42.6 km. A 30 km request is that same line with the west end cut at 长寿路
or 中山公园.

## Ridability — the parts that are surprising

- **The Bund (中山东一路) is not bike-routable.** 外白渡桥 → 外滩南京东路口
  computes to 4.3 km because the router treats the Bund roadway as closed and sends
  you across the river on the 泰公线 ferry. Do not design a route through the Bund
  on bikes; go west along 北苏州路 / 天潼路 instead, and tell the user to walk the
  Bund separately if they want it.
- **苏河步道 and the 杨浦滨江 / 北外滩 public-space promenades are pedestrian-only.**
  Ride the municipal roads: 北苏州路, 光复路, 西苏州路, 天潼路, 杨树浦路, 东大名路.
  State this in the notes — it is the single most likely place a rider gets stopped.
- **The bike profile does route over ferries.** 秦皇岛路码头 is a ferry terminal to
  Pudong; OSRM's ferry crossing between 公平路 and 泰同栈 is what it uses for the
  Bund. A Pudong 东岸滨江 greenway extension is therefore plausible, but confirm
  current operating and bike-boarding rules before promising it.
- **Closed days on the default 苏州河 stops:** 四行仓库抗战纪念馆 and 上海邮政博物馆
  both close Mondays, and 四行仓库 needs advance real-name booking (红途 / 乐游上海
  mini-programs). 上海邮政博物馆 is free.
- **Return legs want a different street.** Outbound along the river and inbound
  inland (西藏北路 → 海宁路 → 溧阳路 → 四平路) is ~7 km and fast; doubling back
  along the riverside road is slower and puts the rider into promenade traffic.
- **Supply gap:** 长风公园 → 大宁灵石公园 is ~9.3 km with no good resupply; water
  up before it. Elsewhere the route passes 杨浦滨江 service stations, 北外滩,
  静安大悦城 and 万象天地.
