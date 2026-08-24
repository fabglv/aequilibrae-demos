# Model inputs

Everything the transit model needs, split into **what you must supply** and **what has a
default you can improve on**. Figures in the "demo" column come from
`notebooks/quito_transit_v2.ipynb`.

---

# A. Strictly necessary

Without these the model either fails to build or produces meaningless results. None of
them has a default.

## A1. GTFS feed — the service

The model's main input: a zip of CSV files describing what runs, where and when. Written
by `write_gtfs()` in A5 of the notebook; in a real project it comes from the agency.

| File | Demo | What the model takes from it |
|---|---|---|
| `agency.txt` | 1 row | Name, timezone. Descriptive only, but required. |
| `routes.txt` | 4 rows | Route id, names, and **`route_type`** — selects the vehicle capacity and decides what gets map-matched. |
| `trips.txt` | 2,278 rows | One row per vehicle run: route, service id, direction. |
| `stops.txt` | 42 rows | Stop id, **lat/lon**, and **`parent_station`** — which platforms belong to one station. |
| `stop_times.txt` | 23,324 rows | The timetable. **Relative** times give run and dwell times; **absolute** times are used only to count trips inside the period. |
| `calendar.txt` *or* `calendar_dates.txt` | 1 row | Which days each service pattern runs. |

Two deviations from the GTFS spec in our feed, both tolerated by AequilibraE:

- **No station rows.** In full GTFS a station is its own row with `location_type = 1`,
  which platforms reference. We set `parent_station` without creating those rows.
  AequilibraE ignores `location_type` entirely and groups on the string alone.
- **`exact_times`** is likewise ignored (commented out of the reader's schema).

## A2. Zoning — where trips start and end

Zone polygons, each with a centroid. Created in A3; a real study would use official TAZ
boundaries.

- Must exist **before** the GTFS import, which assigns each stop to a zone. Requires
  `zoning.refresh_geo_index()` first, or the import fails inside `save_to_disk()`.
- **Zone size sets the modelled walking distance**, because an access connector runs from
  the *centroid* to the stop. It therefore drives the largest component of every journey.
- Demo: a 22 x 10 grid, trimmed and filtered by street length, giving about 187 zones of
  roughly 0.9 km. Access and egress walks come out at a median of about 9 minutes each.
  (The exact count moves by one or two between runs: A2 downloads from OpenStreetMap live, and
  a cell sitting near the `MIN_ROAD_KM` threshold can flip.)

## A3. Demand — an origin-destination matrix

A zone-by-zone matrix of trips. Built synthetically in A7; a real study would use a
survey, smartcard data, or a mode-choice model.

- **Must describe the same window as the service.** A whole day's trips against a
  two-hour service count makes every downstream number wrong by the ratio.
- It is *public-transport* trips only, not all travel.
- Demo: 60,000 trips in the peak, of which about 47,000 have both ends within walking
  distance of a stop.

## A4. Modelled period

A start and end time in seconds from midnight, stored on the project with
`periods.new_period(id, start, end, description).save()`.

- Its only role in the assignment is to decide which trips are counted, which becomes each
  pattern's frequency.
- Demo: 07:00-09:00, `AM_PEAK_ID = 2` (`period_id = 1` is AequilibraE's whole-day default).

## A5. Assignment configuration

Not data, but the run will not work without it:

| Setting | Demo value | Meaning |
|---|---|---|
| `set_time_field` | `trav_time` | Edge cost, in seconds. |
| `set_frequency_field` | `freq` | Vehicles per second, from which waiting time is derived. |
| `set_algorithm` | `os` | Optimal strategies (Spiess & Florian, 1989). |
| `set_skimming_fields` | 7 fields | Which zone-to-zone matrices to produce. Optional, but you get no skims without it. |

---

## A6. Where passengers can change service

A modelling decision with **no default and no rule**: GTFS accepts any grouping you assert
without validating it, and AequilibraE proposes none of its own. Without it the network has no
transfers at all.

The demo uses **two mechanisms**, because the two claims are different:

| | What it asserts | How it is expressed | Demo |
|---|---|---|---|
| **Station** | these platforms are *one physical place* | `parent_station` in `stops.txt`; AequilibraE builds `outer_transfer` and `walking` edges | **3** — El Recreo, La Marín, El Labrador, all within 200 m on foot |
| **Walking transfer** | somebody *could walk* between these two stops | a `walking` edge added to the graph in B2, costed at the routed distance | **10** |

Using `parent_station` for the second is what an earlier version of this notebook did, and it
asserts something false — that a stop 250 m up the road is the same place. GTFS has a file for
transfer rules, `transfers.txt`, but **AequilibraE parses it and never uses it**, so the
connection has to be made on the graph instead.

**Detection uses network walking distance**, not straight-line. The street network from A2 is
routed *undirected* — a pedestrian ignores one-way restrictions — and the ratio of routed to
straight-line distance exposes barriers. The clearest case: `L1_El_Ejido` and `TROLE_Ejido` are
151 m apart in a straight line and **405 m on foot** (x2.68), because El Ejido is a park and the
streets go around it. A straight-line rule called them one station.

Two thresholds, both in B2:

- `SAME_STATION_M` (200 m) — platforms within this walk are one physical place.
- `TRANSFER_WALK_M` (500 m) — beyond this, nobody changes here on foot and the stops stay
  unconnected. Twenty-five pairs within 800 m straight-line fall outside it.

**The audit runs in both directions.** `wide_stations()` in A5 checks the declarations against
the straight-line spread; `audit_interchanges()` in B2 uses the routed walk, so it also catches
a pair that is close on the map but a long way round on foot, and lists the pairs that are
walkable but not connected.

**Why AequilibraE's own walk graph is not used for the routing.** The project registers a walk
mode (`w`) and `build_graphs(modes=["w"])` produces a routable graph, but it respects one-way
direction and 12,053 of the 41,663 walkable links are one-way. Routed that way, two El Recreo
platforms 47 m apart come out *unreachable* and San Francisco to El Playón reads 1,025 m instead
of 369 m. The links and their `distance` come from AequilibraE; the shortest path is taken
undirected with `scipy.sparse.csgraph`.

**Caveat on the metric.** It is the road network, so it has no footpaths, plazas or crossings,
and each stop snaps to the nearest node — a median of 26 m per end, worst 110 m. Read it as an
indicator of detour rather than a measured walk.

An interchange is identified by **how many distinct routes meet there**, not how many platforms
are present — a station grouping two directional platforms of one line has several platforms and
no interchange, which is the ordinary case in an agency feed.

---

# B. Optional — improve realism, defaults exist

## B1. Vehicle capacity

**GTFS has no field for this.** AequilibraE keys it off `route_type`
(`Transit.default_capacities`, as `[seated, total]`):

| `route_type` | Vehicle | Seated | Total |
|---|---|---|---|
| 0 | Tram, streetcar, light rail | 150 | 300 |
| 1 | Subway / metro | 280 | 560 |
| 2 | Rail | 700 | 700 |
| 3 | Bus | 30 | 60 |
| 4 | Ferry | 400 | 800 |
| 5 | Cable tram | 20 | 40 |
| 11 | Trolleybus | 30 | 60 |
| 12 | Monorail | 50 | 100 |
| other | — | 30 | 60 |

Written to the `routes` table at import time, and overridable per route type with
`feed.set_capacities({1: [300, 600], ...})`.

**Nothing reads capacity during assignment** — it is uncapacitated. Capacity only matters
where we compute load-versus-capacity ourselves, in `onboard_by_segment()`. So supplying
real figures improves that check and changes nothing else.

There is a parallel `default_pces` table (passenger-car equivalents, bus = 4.0) used by
`build_pt_preload()` when loading buses onto a road assignment. Not used here.

## B2. Graph construction parameters

All have defaults; the demo changes the first two deliberately.

| Parameter | Demo | Default | Effect |
|---|---|---|---|
| `connector_method` | `overlapping_regions` | nearest stop | Connects a zone to **every** stop in reach, so the boarding stop becomes a choice the assignment makes. |
| `walking_speed` | 1.2 m/s | 1.0 m/s | Scales every access, egress, transfer and platform walk. The default makes all walking 20% longer. Not stored in the saved graph, so it must be set on every rebuild. |
| `walk_time_factor` | 1.0 | 1.0 | Extra multiplier on walking times. |
| `alighting_penalty` | 480 s | 480 s | A routing device that stops the algorithm inventing hop-off-hop-on strategies. It inflates `trav_time`, which is why journey time is summed from its real components instead. |
| `with_walking_edges` | True | False | Platform-to-platform walking inside a station. |
| `with_inner_stop_transfers` | True | False | Change service at the same stop. |
| `with_outer_stop_transfers` | True | False | Change service across a station. |
| `blocking_centroid_flows` | False | — | Left off, which keeps zone numbering aligned with graph node ids. |

## B3. Optional GTFS files

| File | Demo | Status in AequilibraE |
|---|---|---|
| `shapes.txt` | 83 rows | Read. A hint for map matching — links within **20 m** of the shape get their cost cut to a tenth. Ours holds only the stop-to-stop straight line, so it adds nothing beyond `stops.txt`. A real alignment here would materially improve the drawn corridors. |
| `frequencies.txt` | not used | **Fully supported** — expanded into individual trips at import. Would express our timetable in about 16 rows instead of ~26,000. |
| `transfers.txt` | not used | **Parsed but never used.** Transfer times always come from straight-line distance divided by walking speed, so `min_transfer_time` and forbidden transfers have no effect. |
| `fare_attributes.txt`, `fare_rules.txt` | not used | Read, but not used by the assignment. |

## B4. Street network (OpenStreetMap)

Downloaded in A2 with `project.network.create_from_osm()`; 42,253 links in the demo. No
default — without it you simply lose what it provides.

Used for three things, all of them geometry:

1. **Map matching** — finds the roads each bus route runs along, so corridors are drawn on
   real avenues instead of straight lines between stops.
2. **Defining the zoning** — A3 keeps only grid cells containing at least `MIN_ROAD_KM` of
   street, which removes the mountainside.
3. **Map background** — the grey street network behind every map.

**It does not affect any result.** Riding times come from `stop_times`, and every walking
leg is a straight line divided by the walking speed. Remove the OSM step and the loads,
journey times and skims are identical — only the maps get worse and the zoning keeps its
empty cells.

---

# C. Derived, not inputs

Easy to mistake for data:

- **Patterns** — route + direction + stop sequence. The demo's 4 routes become 8 patterns.
- **Frequencies** — counted from the trips falling inside the period.
- **Interchanges** — emergent, wherever transfer edges join two different routes.
  Note the *derivation* is automatic but the *grouping* it works from is declared by hand (see A6).
- **The transit graph** — boarding, on-board, dwell, alighting, transfer, walking and
  connector edges, all built from the feed.
- **Walking times** — straight-line distance over walking speed. Never routed on the
  street network, never read from a file.
- **Corridor alignment** — inferred by map matching: shortest path *by distance* between
  consecutive stops, biased toward the declared shape.

---

# D. Minimum viable input set

To build and solve a transit model you need exactly four things:

1. A **GTFS feed** with `agency`, `routes`, `trips`, `stops`, `stop_times`, and
   `calendar` (or `calendar_dates`).
2. A **zoning** with centroids.
3. A **demand matrix** for the modelled window.
4. A **period** declaration.

Plus, if the network is to have any transfers at all, **interchange declarations** (A6) - which have no default.

Everything in section B has a default or is optional. The two defaults most worth
replacing with real data are **vehicle capacity** (or the capacity map is only as good as
a route-type guess) and **walking speed** (which scales the largest component of every
journey).
