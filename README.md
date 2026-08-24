# Quito Transport Models — Interactive Scenarios

Two self-contained [AequilibraE](https://www.aequilibrae.com/) notebooks that build transport
models of Quito, Ecuador from scratch — one for road traffic, one for public transport.

## Road model — `notebooks/quito_scenarios_demo.ipynb`

- **Part A — baseline.** Pulls Quito's road network from OpenStreetMap (~42,000 links), lays a
  12 × 6 grid of Traffic Analysis Zones over it, sets speed and capacity per road class, and
  generates a synthetic O-D matrix of ~25,000 peak-hour car trips.
- **Part B — solve.** Static user-equilibrium assignment (BPR, bi-conjugate Frank-Wolfe), cached as
  the reference case, plus the comparison table and interactive
  [folium](https://python-visualization.github.io/folium/) maps.
- **Part C — scenarios.** Close a road, change capacity/speed, change demand, add a new road —
  each one function call, compared against the baseline. Only the new-road case touches the
  database, and it undoes itself.

## Transit model — `notebooks/quito_public_transit_demo.ipynb`

- **Part A — baseline.** Writes a **GTFS feed** for Quito's trunk network — Metro Línea 1, the
  Trolebús and Ecovía BRT corridors, and a northern feeder — imports it, map-matches the bus
  routes onto the streets, lays a 22 × 10 grid of zones, and generates a synthetic O-D matrix of
  60,000 peak-hour transit trips.
- **Part B — solve.** Frequency-based **hyperpath assignment** (optimal strategies, Spiess &
  Florian 1989), plus who the network reaches, where the passengers are, and how full each service
  runs.
- **Part C — scenarios.** Run more vehicles on a line, or add a new line by editing the feed and
  importing it again. The second rewrites the transit database and undoes itself.

## Running it

Requires **Python 3.12** and an internet connection for the OSM download.

```bash
pip install -r requirements.txt
jupyter lab notebooks/quito_scenarios_demo.ipynb        # road
jupyter lab notebooks/quito_public_transit_demo.ipynb   # transit
```

Run the cells in order — a few minutes each, most of it the download in cell A2.

`data/` (the AequilibraE SQLite projects) and `outputs/` (the interactive HTML maps) are committed
empty and regenerated from scratch on every run, so neither is tracked by git.

## Reading the maps

Maps render inline and are also written to `outputs/maps/` for the road model and
`outputs/transit_maps/` for the transit model. Hover anything for its figures; every panel ends
with an identifier you can look up or feed back into a scenario function.

## Caveats

**The numbers are illustrative, not calibrated.** Both O-D matrices are synthetic and the zoning is
a grid rather than official TAZ boundaries. The transit timetable is generated from a headway and a
commercial speed rather than copied from a published schedule, and its assignment is
**uncapacitated** — a segment carrying more passengers than the vehicles have places is reported
without complaint, so comparing load against capacity is left to the reader.
