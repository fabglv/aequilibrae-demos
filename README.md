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
jupyter lab notebooks/TAZ_OD_demo.ipynb                 # zones and O-D matrix
```

Run the cells in order — a few minutes each, most of it the download in cell A2.

`data/` and `outputs/` are committed empty and neither is tracked by git. The AequilibraE
projects and the HTML maps regenerate on every run, and the road network downloads once and is
then cached in `data/road_network/`.

**`TAZ_OD_demo.ipynb` needs data this repository does not carry.** It is official DMQ material, not
something the notebook can download. Put it in `data/gov_data/`:

| what | where it goes |
|---|---|
| census sectors | `sector_censal_2022_inec_q/sector_anonimizado_a.shp` |
| barrio layer | `BARRIO_REF/BARRIO_REF.shp` |
| population projections | `03_Resultados_proyeccion2023_2035.xlsx` |
| land-use plan | `ba003_uso_suelo_edificabilidad_a/ba003_uso_suelo_edificabilidad_a.shp` |
| cadastre | `BLOQUE_CONSTRUCTIVO/BLOQUE_CONSTRUCTIVO/catastro.gdb` |

Each is checked where it is first read, and the error names the path that is missing.

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
