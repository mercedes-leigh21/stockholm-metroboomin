# Observable import — Stockholm × Tunnelbana

Files in this folder:

| File | What for |
|---|---|
| `stations.json` | 100 Tunnelbana stations (Wikidata SPARQL `P16=Q272926`). Authoritative. |
| `population.json` | Stockholm city + county population 1900–2024 (Wikipedia decades + SCB API annual). |
| `deso_stockholm.json` | 1287 DeSO neighborhood polygons in Stockholm county + population per area 2010–2023 (SCB WFS + SCB API). ~955 KB simplified GeoJSON. *(Currently unused by the page — kept for future choropleth or density features.)* |
| `catchment.json` | Per-DeSO 2023 population + `first_served` year (year of first Tunnelbana station to come within 1 km of the DeSO centroid). 71 KB. Used by the catchment % callout. |
| `stockholm-tunnelbana.md` | **Observable Framework** page — markdown with fenced ```js cells. |
| `stockholm-tunnelbana.ojs` | **Observable cloud notebook** source — `//// CELL:` blocks for paste-as-cell import. |

The notebook renders a side-by-side dual-panel view: Leaflet map (left, ~520 px tall) + two `Plot` charts (right) — population area chart and cumulative-stations step chart, both reacting to a year slider.

---

## Observable Framework (recommended)

```bash
# in your Framework project
cp stations.json population.json src/
cp stockholm-tunnelbana.md src/

npm run dev   # or: observable preview
```

The page uses standard Framework conventions (`FileAttachment` for data, `Plot` and `Inputs` from the standard library). No extra dependencies. Leaflet loads from CDN.

---

## Observable cloud notebook (observablehq.com)

1. Create a new notebook.
2. Open `stockholm-tunnelbana.ojs` and paste each `//// CELL:` block as one notebook cell, in order.
3. Upload `stations.json` and `population.json` via the file panel on the right.
4. The cells are reactive — drag the year slider and the map, stat cards, and both charts re-evaluate automatically.

**Required cell order** (dependencies):
1. Markdown cells: `title`, `subtitle`
2. Data: `stations`, `popData`, `cityPop`
3. Library: `L` (Leaflet)
4. CSS injection cell
5. Constants: `lineColors`, `annotations`
6. Helpers: `fmt`, `popAt`, `stationsAt`
7. Input: `viewof year` (the master slider)
8. Stat cards (HTML)
9. Map: `tbMap`
10. Reactive `activeLayer` + side-effect attach
11. Population chart
12. `stnSeries`
13. Stations chart
14. Footer

---

## How the data was sourced

**Stations** — Wikidata SPARQL:
```sparql
SELECT DISTINCT ?station ?stationLabel ?opening ?lat ?lon ?lineLabel WHERE {
  ?station wdt:P16 wd:Q272926 .            # transport network = Stockholm Metro
  OPTIONAL { ?station wdt:P1619 ?opening } # date of official opening
  OPTIONAL { ?station wdt:P571  ?inception }
  OPTIONAL { ?station p:P625/psv:P625 ?c .
             ?c wikibase:geoLatitude ?lat . ?c wikibase:geoLongitude ?lon }
  OPTIONAL { ?station wdt:P81 ?line .
             ?line rdfs:label ?lineLabel . FILTER(LANG(?lineLabel)="sv") }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "sv,en" }
}
```
Result deduped by station QID, prefer `P1619` over `P571`, drop rows missing coords or dates, strip " tunnelbanestation" from labels. Yields **100 stations** with verified opening dates.

**Population, 1968–2024** — SCB PxWeb API:
```
POST https://api.scb.se/OV0104/v1/doris/en/ssd/BE/BE0101/BE0101A/BefolkningNy
{
  "query": [
    {"code": "Region",      "selection": {"filter": "item", "values": ["0180"]}},
    {"code": "Civilstand",  "selection": {"filter": "all",  "values": ["*"]}},
    {"code": "Alder",       "selection": {"filter": "all",  "values": ["*"]}},
    {"code": "Kon",         "selection": {"filter": "item", "values": ["1","2"]}},
    {"code": "ContentsCode","selection": {"filter": "item", "values": ["BE0101N1"]}}
  ],
  "response": { "format": "json" }
}
```
Sum across age × civilstånd × kön. **Watch out:** the table includes a synthetic age value `'tot'` that double-counts every person — filter it out before summing, or your totals will be exactly 2× too high.

**Population, 1900–1960** — Wikipedia [City of Stockholm (city municipality)](https://en.wikipedia.org/wiki/City_of_Stockholm_(city_municipality)) demographic table, decade snapshots. Note: Stockholm boundaries changed slightly when the city became Stockholm Municipality in 1971, but the discontinuity is small (~5k of ~750k).

---

## Tweaks worth considering

- **Use a Mapbox / CartoDB Dark Matter basemap** — drop the `filter:` style on `.leaflet-tile-pane` if you do; it's only there to dim the bright OSM defaults.
- **Switch the Leaflet map to deck.gl** — the data shape (`{lat, lon, year, lines}`) maps directly to a `ScatterplotLayer` with `getFillColor` keyed off `year` vs `currentYear`. More animation-friendly.
- **Add Stockholm county on the population chart** — `population.json` already includes the `county` array. A second area in a different shade would show the suburbanization split: county growing while city shrank 1960–1981.
- **Add line geometry between stations** — needs OSM/Trafikverket fetch (see `../SPEC.md`). Once you have it, `Plot.geo` or a Leaflet polyline layer would draw it nicely.
- **Add real annotation labels** (not just tick marks) — `Plot.text` with the `annotations` array. The current ticks are intentional minimal-chrome; if you want labeled milestones, swap them in.
