# Project Task Checklist

Living checklist for the Sweden Railway Growth Map project. Check items off as you finish.

---

## Phase 0 — Decisions (do these first)

- [ ] Decide on time range: 1950→today (mostly contraction) vs 1850→today (full growth arc). See [SPEC.md §7](SPEC.md).
- [ ] Decide whether to include closed lines. (Strong recommendation: yes — without them, 1950→today is uninteresting.)
- [ ] Decide whether to include city trams (Gothenburg, Norrköping inner-city).
- [ ] Decide on coloring strategy: by category (default) vs by operator.
- [ ] Decide on data-effort budget: full national coverage (multi-day) vs ~30 most-significant-lines (1–2 days).
- [ ] Get a Google Maps JavaScript API key (or stick with the Leaflet/OpenStreetMap prototype basemap — no key needed).

## Phase 1 — Prototype (see PROTOTYPE files in this repo)

- [x] Build a static HTML prototype with year slider + map + sample dataset to validate visualization style.
- [x] Curate hand-coded sample dataset (~20 segments) covering the visually dramatic moments: Tunnelbana buildup, narrow-gauge closures, Öresundsbron, Botniabanan, Citybanan.
- [x] Implement "easy to understand" visualization: bold lines, ghost layer of all-time network, color-coded just-opened / just-closed flashes.
- [ ] Get your eyes on the prototype and decide what to keep / change. Open `index.html` in a browser.

## Phase 2 — Data outreach (parallel-safe, low effort, high upside)

- [ ] Email Prof. Jordi Martí-Henneberg (Universitat de Lleida) asking if his European historical railway GIS dataset (1840–2010) covers Sweden adequately and is shareable. See [DATA_SOURCES_MEMO.md §3](DATA_SOURCES_MEMO.md).
- [ ] Email Sveriges Järnvägsmuseum (Gävle) asking if they have or know of a section-level Swedish railway opening-date dataset.
- [ ] Email järnväg.net maintainers asking permission to cite/republish their opening-date data.

## Phase 3 — Real geometry pipeline

- [ ] Write `scripts/fetch-osm.js` — Overpass query for all Sweden railway ways (`rail`/`subway`/`light_rail`/`tram`/`narrow_gauge`/`abandoned`/`disused`). Output: raw GeoJSON.
- [ ] Write `scripts/segment.js` — split ways at junctions and station boundaries into proper "bandel"-aligned segments.
- [ ] Write `scripts/simplify.js` — coordinate precision trim to 5 decimals, optional Douglas-Peucker for country-zoom level.
- [ ] (Optional) Write `scripts/fetch-trafikverket.js` — Lastkajen download for current heavy rail, cross-check against OSM.
- [ ] Verify output: total ways ~33,185 from OSM, simplified file <2 MB gzipped.

## Phase 4 — Real date pipeline

- [ ] Write `scripts/fetch-wikidata.js` — SPARQL query for all 65 Swedish lines with inception dates. Output: line → opening year CSV.
- [ ] Write `scripts/extract-runeberg.js` — for each volume of "Statens järnvägar 1856-1906" and "1906-1931" on Project Runeberg, fetch the OCR text and use an LLM to extract structured per-line opening dates.
- [ ] Write `scripts/extract-wikipedia.js` — for each Wikipedia article matching a line in our segment dataset, fetch and LLM-extract section-by-section opening dates.
- [ ] Write `scripts/join-dates.js` — merge all date sources into a single `segment_id → opened, closed` mapping. Track provenance per segment.
- [ ] Generate manual-review CSV: segments still missing dates after auto-extraction, sorted by line + length. Fill in by looking at järnväg.net and historiskt.nu.
- [ ] Sanity-check pass: every segment has either `opened` or is flagged "unknown".

## Phase 5 — Validation

- [ ] Write `scripts/validate-against-scb.js` — for years 1862–1910, sum our dataset's segment lengths in service that year and compare against SCB BiSOS L published network length.
- [ ] Investigate any year where discrepancy exceeds 2%.
- [ ] Manually spot-check 20 well-known segments (Tunnelbana red line opening, Öresundsbron, etc.) against Wikipedia.

## Phase 6 — Production app

- [ ] Convert prototype to use Google Maps JS API (or keep Leaflet — see Phase 0 decision).
- [ ] Wire `data/segments.geojson` (full real dataset) into the existing prototype code.
- [ ] Add category filter checkboxes (Heavy rail / Metro / Light rail / Closed).
- [ ] Add "by operator" coloring toggle (optional).
- [ ] Performance test with full ~5,000 segments. If sluggish, switch to canvas-backed rendering.
- [ ] Add tooltips / click info windows with segment details + Wikipedia link.
- [ ] Add legend, year-info panel ("opened this year: …", "closed this year: …", "total km in network").

## Phase 7 — Polish & ship

- [ ] Mobile responsive tweaks (or document desktop-only).
- [ ] Error states: API down, slow network, no JS.
- [ ] Set up `config.example.js` and `.gitignore` for API key.
- [ ] Deploy to GitHub Pages.
- [ ] Set up Google Maps API billing alerts.
- [ ] Write a short "About this map" page: data sources, methodology, known gaps, link to GitHub.
- [ ] Contribute back: upload our segment dataset to OpenHistoricalMap as a community contribution.

---

## Definitely-out-of-scope (see [SPEC.md §6](SPEC.md))

City trams, museum railways, industrial sidings, ridership data, mobile-first design, i18n, historical basemap tiles.
