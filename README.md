# Quito Road Model — Interactive Scenarios

A self-contained [AequilibraE](https://www.aequilibrae.com/) notebook that builds a macroscopic
road-traffic model of Quito, Ecuador from scratch.

## What it does

- **Part A — baseline.** Pulls Quito's road network from OpenStreetMap (~42,000 links), lays a
  12 × 6 grid of Traffic Analysis Zones over it, sets speed and capacity per road class, and
  generates a synthetic O-D matrix of ~25,000 peak-hour car trips.
- **Part B — solve.** Static user-equilibrium assignment (BPR, bi-conjugate Frank-Wolfe), cached as
  the reference case, plus the comparison table and interactive
  [folium](https://python-visualization.github.io/folium/) maps.
- **Part C — scenarios.** Close a road, change capacity/speed, change demand, add a new road —
  each one function call, compared against the baseline. Only the new-road case touches the
  database, and it undoes itself.

## Running it

Requires **Python 3.12** and an internet connection for the OSM download.

```bash
pip install -r requirements.txt
jupyter lab notebooks/quito_scenarios_demo.ipynb
```

Run the cells in order — a few minutes total, most of it the download in cell A2.

`data/quito_scenarios_project/` (the AequilibraE SQLite project) and `outputs/maps/` (seven
interactive HTML maps) are committed empty and regenerated from scratch on every run, so neither is
tracked by git.

## Reading the maps

Maps render inline and are also written to `outputs/maps/`: the zone system, zones with demand,
baseline V/C, and one per scenario. Hover any street for its figures; every panel ends with the
`link id` so anything you see can be looked up or fed back into a scenario function.

## Caveats

**The numbers are illustrative, not calibrated.** The O-D matrix is a reproducible random draw, the
speeds and capacities are class defaults, and the zoning is a synthetic grid rather than official
TAZ boundaries.
