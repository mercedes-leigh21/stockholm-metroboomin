# Stockholm — A century in motion

Interactive visualization of how the Stockholm Tunnelbana grew between 1900 and 2024, alongside Stockholm city + county population, with a 1 km **catchment** metric showing the share of residents who actually live within walking distance of a metro station.

Built with vanilla HTML + Leaflet + D3 for the standalone app, plus parallel ports for **Observable Framework** and the **Observable cloud notebook**.

---

## Quick start

```bash
# Serve at http://localhost:8090
python3 -m http.server 8090
# Then open http://localhost:8090 in a browser
```

The page must be served over HTTP, not opened directly via `file://` — the Flatpak document portal sandboxes `file://` access and blocks the JSON fetches.

## What you see

- **Map (left):** Leaflet basemap dimmed; 100 Tunnelbana stations colored by line, with newly-opened stations flashing yellow as you scrub the year. Two map toggles in the bottom-right:
  - **Show metro arrival year** — choropleth painting each of 1,287 DeSO neighborhoods by the year metro first arrived within 1 km.
  - **Show 1 km catchment** — translucent gold rings around each open station.
- **Right panel:** big year display, three stat cards (city pop, county pop, station count), a gold callout with the live **% within 1 km of a Tunnelbana station** (and the count of people that represents), and two D3 charts — Greater Stockholm population stacked area (city + county) and cumulative-stations step.
- **Slider (bottom):** 1900 → 2024 with a play button.

## The data

| Source | What | Coverage |
|---|---|---|
| **Wikidata SPARQL** (`P16=Q272926`) | Tunnelbana stations: name, opening year, coords, lines | 100 stations |
| **SCB API** `BE/BE0101/BE0101A/BefolkningNy` | Stockholm kommun + Stockholms län annual population | 1968 → 2024 |
| **Wikipedia** "City of Stockholm (city municipality)" | Stockholm city historical population | 1900 → 1960 (decades) |
| **SCB WFS** `stat:DeSO_2018` | DeSO neighborhood polygons (Stockholm county) | 1287 polygons |
| **SCB API** `BE/BE0101/BE0101Y/FolkmDesoAldKon` | Per-DeSO population by year | 2010 → 2023 |

All data files live in [`data/`](data/) (and a duplicated copy in [`observable/`](observable/) for the FileAttachment-style imports). Provenance comments are inline in the JSON.

### Catchment computation

For each DeSO, `first_served = min(station.year for station whose haversine distance from DeSO centroid is ≤ 1 km)`. The catchment % is `sum(2023 pop of DeSOs where first_served ≤ year) / total 2023 pop`. Holding the denominator at the 2023 population is intentional — the metric varies only with network expansion, not with population shifts, giving a clean counterfactual: *"how much of today's Stockholm would have been served by year Y's network?"*

The script that produces `data/catchment.json` from `data/deso_stockholm.json` + `observable/stations.json` is documented inline in `observable/README.md`.

## File map

```
index.html                      # the standalone app — vanilla HTML, Leaflet, D3, all CSS + JS + station data inlined
data/                           # runtime fetches for index.html
  ├── population.json           # city + county population time series
  ├── deso_stockholm.json       # 1287 DeSO polygons (Stockholm county), simplified
  └── catchment.json            # per-DeSO 2023 pop + first_served year — drives the catchment % + arrival-year choropleth
observable/                     # parallel ports for Observable Framework + cloud notebook
  ├── stockholm-tunnelbana.md   # drop into Framework src/
  ├── stockholm-tunnelbana.ojs  # paste cell-by-cell into a cloud notebook
  ├── stations.json             # canonical Wikidata-extracted stations (also inlined in index.html)
  ├── population.json           # = data/population.json
  ├── deso_stockholm.json       # = data/deso_stockholm.json
  ├── catchment.json            # = data/catchment.json
  └── README.md                 # import paths + the SPARQL/SCB query that produced the data
SPEC.md                         # original product spec (paper trail; predates the realised app)
DATA_SOURCES_MEMO.md            # findings from the data-source verification pass
TASKS.md                        # phase-by-phase planning checklist
```

The two folders intentionally duplicate the data files — `data/` is what `index.html` fetches over HTTP, `observable/` is what `FileAttachment` resolves in Framework / cloud notebooks. They're regenerated together; keep them in sync.

## Observable port

Two parallel ports of the same visualization in [`observable/`](observable/):

- `stockholm-tunnelbana.md` — drops into an Observable Framework `src/` directory and works immediately. Uses `Plot.plot` for the charts, `Inputs.range`/`Inputs.toggle` for inputs, and `FileAttachment` for data.
- `stockholm-tunnelbana.ojs` — same code annotated with `//// CELL: name` separators so you can paste each block into a fresh Observable cloud notebook in order.

See [observable/README.md](observable/README.md) for both import paths.

## Methodological honesty

A few things worth being explicit about:

- **DeSO centroid distance is approximate** for the catchment metric — a long thin DeSO might extend beyond 1 km on its tail end even if its centroid is within range. DeSOs are designed for roughly equal population (~1500–2000 each), so this approximation is fine for visualization purposes but isn't survey-grade.
- **Catchment denominator is fixed at 2023** to keep the metric a clean network counterfactual. Using each year's actual population would conflate network expansion with population shifts.
- **Stations dataset is Tunnelbana only.** Pendeltåg, Roslagsbanan, Saltsjöbanan, Lidingöbanan, and Tvärbanan are excluded. Adding them would push the catchment % from 39 % to roughly 70–80 %.
- **Boundary discontinuity in 1971** when Stockholm city became Stockholm Municipality with slightly expanded borders (~5 k of ~750 k people). Visible as a tiny step in the city pop curve.
- **SCB DeSO `'tot'` age bucket double-counts** — when summing across age groups for total population, you must filter `Alder !== 'tot'` or your numbers will be exactly 2× too high. Documented inline in the data-extraction notes.

## Sources

- [Wikidata Q272926](https://www.wikidata.org/wiki/Q272926) — Stockholm Metro
- [SCB API documentation](https://www.scb.se/en/services/open-data-api/)
- [SCB DeSO open geodata](https://www.scb.se/en/services/open-data-api/open-geodata/open-data-for-deso--demographic-statistical-areas/)

---

*Generated as a hobby project. Data is open and freely redistributable; this code is MIT.*
