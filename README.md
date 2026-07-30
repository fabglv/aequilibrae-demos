# Quito Road Model — Interactive Scenarios

A self-contained [AequilibraE](https://www.aequilibrae.com/) notebook that builds a macroscopic
road-traffic model of Quito, Ecuador from scratch, solves it, and then runs four kinds of
*what-if* scenario on it.

It is written for someone who has **never used AequilibraE before**. Every step is explained in
plain language first, and the modelling logic sits in a handful of small, reusable functions —
`build_graph(...)`, `solve_car(...)`, `compare(...)` — so the same code could later sit behind a
graphical interface with one button per scenario.

The one idea behind the whole notebook: **a scenario is an edit to the inputs, followed by
re-running the model and comparing the numbers.**

## What it does

**Part A — build the baseline.** Downloads Quito's road network from OpenStreetMap (~42,000
links), lays a 12 × 6 grid of Traffic Analysis Zones over it, attaches each zone centroid to the
network, assigns a speed and capacity to every link from its road class, and generates a
synthetic origin–destination matrix of ~25,000 peak-hour car trips.

**Part B — solve it.** Runs a static user-equilibrium assignment (BPR volume-delay function,
bi-conjugate Frank-Wolfe) and caches the result as the reference case. Also builds the reporting
helpers: a comparison table and five interactive [folium](https://python-visualization.github.io/folium/)
maps.

**Part C — four scenarios**, each one function you call and compare against the baseline:

| Scenario | What changes | Touches the database? |
|---|---|---|
| Close a road | links removed from routing | No (in-memory) |
| Change capacity or speed | `capacity` / `travel_time` of some links | No (in-memory) |
| Change demand | the O-D matrix | No (in-memory) |
| Add a new road | a link is inserted, then deleted again | Briefly — undone afterwards |

## What kind of model this is

A **static assignment**, not a microsimulation — the distinction matters and the notebook opens
with a section on it. There is no clock and there are no individual vehicles: the model solves for
a single steady-state equilibrium representing one period. Every output (link volumes, V/C ratios,
total vehicle-minutes) is a **period average**, not a time-varying picture of jams forming and
clearing.

One consequence to keep in mind: link capacity is stored in vehicles **per hour**, so the demand
matrix must be on the same basis — peak-hour trips, not a daily total.

## Running it

Requires **Python 3.12** and an internet connection for the OpenStreetMap download.

```bash
pip install -r requirements.txt
jupyter lab notebooks/quito_scenarios_demo.ipynb
```

Then run the cells in order. The notebook works whether the kernel starts in the repository root
or in `notebooks/`. The OSM download in cell A2 is the slow step (one to two minutes); everything
after it is fast. A full run takes a few minutes.

These two directories are committed empty — running the notebook fills them:

```
data/quito_scenarios_project/   the AequilibraE project (SQLite) it builds
outputs/maps/                   seven interactive HTML maps
```

Both are regenerated from scratch on every run and are therefore not tracked by git.
`outputs/maps/` is cleared at the start of each run, so it only ever holds the current set.

## The maps

The notebook renders these inline and also writes each one to `outputs/maps/`, so you can open
them in a browser. Every map shares a single hover-panel design; hover any street for its figures,
and every panel ends with the `link id` so anything you see can be looked up in the database or
fed back into a scenario function.

| File | What it shows |
|---|---|
| `0_zones_and_centroids.html` | The zone system, from geometry alone — no demand, no results |
| `1_zones_with_demand.html` | The same, enriched with trips per zone and the connectivity check |
| `2_baseline_congestion.html` | Baseline V/C by link — blue free-flowing, red over capacity |
| `3_close_road.html` | Where traffic went when the busiest artery was cut |
| `4_capacity_change.html` | Change in *congestion* after widening three corridors |
| `5_demand_change.html` | Where a new commuter flow between two zones ends up |
| `6_new_road.html` | Who moves onto a new express link, and which streets it relieves |

Two conventions worth knowing before reading them:

- **Grey means "below the highlight threshold", not "unchanged".** An equilibrium assignment nudges
  thousands of links by a vehicle or two, which is solver noise rather than a result. Each map
  prints how many links stayed grey and what share of the total change they represent, and states
  the threshold in its legend.
- **The widening scenario is mapped by congestion change, not volume change.** A widened road
  usually attracts *more* traffic, so a volume map would paint a successful project red.

## Known and expected output

Four of the 72 zones cover roadless mountainside and stay flagged `NOT CONNECTED`. This is
deliberate, not a bug: rather than stretching a connector for kilometres and quietly loading
streets that should be empty, the notebook leaves those zones unattached, reports them, and prints
how much of the matrix can actually be assigned. Section B3 explains the reasoning and the repair
logic in full.

AequilibraE also emits `UserWarning: Found centroids not present in the graph!` for exactly those
zones. It is expected and harmless here.

## Caveats

**The numbers are illustrative, not calibrated.** Two inputs would have to be replaced before any
result here means anything about real Quito traffic:

- the **O-D matrix is synthetic** — a reproducible random draw, not survey or gravity-model demand;
- the **link speeds and capacities are defaults by road class**, not measured values.

Calibrating those against Quito's real bottlenecks is where the actual modelling work lies. The
zoning is likewise a synthetic grid; a real study would use official TAZ boundaries.

## Licence

None. This is a demonstration of what AequilibraE can do and a base to build on, not a package to
depend on, so it is published without a licence and default copyright applies. Ask if you would
like to reuse any of it.
