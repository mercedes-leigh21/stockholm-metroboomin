# Sweden Railway Growth Map — Specification

An interactive HTML page that visualizes how the Swedish railway network has changed from 1950 to today, on top of Google Maps.

---

## 1. Goal

Let a user scrub through time (1950 → present) and see how Sweden's rail network has evolved year by year. Each segment appears on the map at the year it opened, and disappears at the year it closed.

**Important context — read before agreeing to scope:** the Swedish rail network *peaked* around 1938 at roughly 16,800 km. Since 1950 the story has been predominantly **contraction**: hundreds of rural branch lines and narrow-gauge railways were closed in the 1950s–80s. Net new construction since 1950 is concentrated in a relatively small number of high-profile projects:

- Stockholm Tunnelbana (1950–1994, plus the upcoming extensions)
- Gothenburg & Malmö light rail/tram extensions
- Öresundsbron + connecting lines (2000)
- Botniabanan (2010)
- Citytunneln Malmö (2010), Citybanan Stockholm (2017), Hallandsås tunnel (2015)
- Various electrifications and double-track upgrades (geometry unchanged but operationally significant)

So if the goal is "watch the network *grow*," the more visually compelling time window is actually pre-1950, or 1850–today. If the goal is "see how the network has *changed* (mostly shrunk, with some new corridors)," 1950→today is right. **Worth confirming which story you want to tell.**

**Scope of "railway":**
- Heavy rail managed by Trafikverket / operated by SJ, MTR, SL Pendeltåg, Skånetrafiken Pågatåg, Västtrafik, Norrtåg, etc.
- Privately-owned / regional rail: Inlandsbanan, Arlanda Express, etc.
- Metro: Stockholm Tunnelbana.
- Light rail / tram-train: Tvärbanan, Roslagsbanan, Saltsjöbanan, Lidingöbanan, Göteborgs spårvägar (light rail extensions only, not the dense city tram grid), Lund–ESS tramway.
- Excluded by default: city tram networks (Gothenburg/Norrköping inner-city grid), industrial sidings, museum railways, freight-only short spurs.

**Geographic scope:** The whole of Sweden (roughly 55.3°N–69.1°N, 11.0°E–24.2°E).

---

## 2. User Experience

### Page layout
- **Map (full-screen):** Google Maps base map, centered on Sweden (~62.5°N, 16.5°E), default zoom ~5 — fits the full country. User can zoom in to Stockholm, Malmö, etc.
- **Time slider (bottom overlay):** Range 1950 → current year, step = 1 year. Selected year displayed prominently.
- **Play/pause button:** Auto-advances one year per ~500 ms.
- **Legend (top-right overlay):** Color key — see §2.1 below for the strategy, since "color per operator" gets unwieldy nationally.
- **Year info panel (top-left overlay):** Year, total track-km in network at that year, count of "Opened this year" / "Closed this year" with a short list (top 5 by length).
- **Filter toggles (in legend):** Checkboxes to show/hide categories — Heavy rail, Metro, Light rail/tram, Closed lines. Useful because zoomed out to all of Sweden, the metro/light rail dots are invisible noise; zoomed into Stockholm, the user wants them prominent.

### 2.1 Coloring strategy
With ~20+ operators nationally, per-operator colors become hard to read. Recommended strategy:

- **By category, not operator:** Heavy rail = dark grey, Metro = blue, Light rail/tram-train = orange, Closed-since-1950 = faded red.
- **Highlight overrides:** Newly-opened-this-year segments flash bright green for ~2 seconds; newly-closed-this-year segments flash red.
- **Optional:** a "color by operator" toggle for users who want SJ blue / MTR purple / etc., but default is by category.

### Interaction
- Dragging the slider redraws the visible network instantly.
- Hovering a segment shows a tooltip: line name, opened year, closed year (if any), operator.
- Clicking a segment opens an info window with description and Wikipedia link.
- Segments that opened/closed in the currently selected year are visually highlighted briefly after a year change.

---

## 3. Data Model

Each track segment is one GeoJSON `LineString` feature:

```json
{
  "id": "botniabanan-ornskoldsvik-husum",
  "name": "Örnsköldsvik ↔ Husum",
  "line": "Botniabanan",
  "operator": "Trafikverket",
  "category": "heavy_rail",
  "opened": 2010,
  "closed": null,
  "gauge_mm": 1435,
  "electrified": true,
  "source": "https://sv.wikipedia.org/wiki/Botniabanan",
  "color": null
}
```

- `opened` (required, int): year the segment entered service.
- `closed` (nullable int): year decommissioned. `null` = still in service.
- `category` (required): `heavy_rail` | `metro` | `light_rail` | `tram_train` | `narrow_gauge`. Drives default color and the filter checkboxes.
- `gauge_mm`: useful for visualizing the narrow-gauge contractions (most narrow-gauge lines closed 1950–80).
- `electrified`: optional, only useful if we add an "electrification year" overlay later.
- Geometry: `LineString` in WGS84 (EPSG:4326).

**Visibility rule:** segment visible at year Y iff `opened <= Y && (closed == null || closed > Y)`.

### File layout
- `data/segments.geojson` — full FeatureCollection.
- `data/operators.json` — operator → display-name + default color.
- Dataset size estimate: **2,000–5,000 segments** for nationwide coverage. Larger than Stockholm-only but Google Maps Data layer handles this fine if rendered as polylines (not symbols).

---

## 4. Data Sourcing — significantly harder at national scale

We need two things per segment: **geometry** (the line on the map) and **dates** (opened/closed years). They don't usually come from the same source. Below is a wide brainstorm of options, then a recommended pipeline.

> ⚠️ I am reasonably confident these institutions and datasets exist, but I have not verified every URL, format, or licensing condition for this spec. Any path we pick should start with a 30-minute "does this dataset actually exist and is it usable" verification before we commit to it.

### 4.1 Government & official statistics

- **SCB — Bidrag till Sveriges officiella statistik (BiSOS), Serie K "Kommunikationsväsendet"** *(probably the single best primary source)*. From 1856 onwards, SCB's predecessor published annual statistics on the railway sector. The volumes contain tables of: lines opened that year, length added, gauge, owner. Continued post-1912 as "Sveriges officiella statistik" (SOS). Most of this is digitized as searchable PDFs on scb.se and at Project Runeberg (runeberg.org). **Implication:** you can OCR/parse these tables to get a *year-by-year ground-truth* of how many km of network existed, broken down by line — a fantastic validation set even if the geometry comes from elsewhere.
- **Trafikverket — Lastkajen / Nationell järnvägsdatabas (NJDB).** The current operational rail network as open GIS data (heavy rail only). No historical layers, but authoritative for current geometry.
- **Trafikanalys** (the modern transport-statistics agency, split off from SIKA). Modern aggregate stats; less useful per-segment but good for headline numbers.
- **Lantmäteriet — Historiska kartor.** Digitized historical map series: *Generalstabskartan* (1820s–1970s), *Häradsekonomiska kartan* (1860s–1930s), *Ekonomiska kartan* (1935–1978). Railways are clearly drawn on these. For lines we can't otherwise date, tracing from a dated map series gives at least an upper-bound year. Many sheets are open-licensed (CC0/CC-BY).
- **Riksarkivet (Swedish National Archives).** Holds the SJ company archive (post-1856) and earlier private railway companies. Includes operational records, opening protocols, line maps. Mostly paper/scanned, not structured data, but the definitive source if a Wikipedia article disagrees with järnväg.net.
- **Krigsarkivet (Military Archives).** Historical military maps often show railways with strategic accuracy. Useful for cross-checking 1940s–60s state.
- **Riksantikvarieämbetet (RAÄ).** Cultural heritage register — abandoned rail infrastructure (stations, embankments) is sometimes registered as fornlämning, with dates.

### 4.2 Universities & academic datasets

- **Jordi Martí-Henneberg (Universitat de Lleida) — European Historical Railway GIS, 1840–2010.** *(High-value if accessible.)* This research group has published pan-European historical railway shapefiles with opening dates. If it covers Sweden adequately, it could be 80% of the work done. Likely available via journal supplementary material (*Journal of Transport Geography*, *Historical Methods*) or by emailing the group directly. **Worth one email to find out.**
- **KTH (Kungliga Tekniska Högskolan).** Department of Transport Science / division of Transport Planning have published research using Trafikverket data. Some PhD theses on Swedish railway history may have associated datasets in DiVA (Sweden's academic publication portal, diva-portal.org).
- **Lunds universitet — Department of Economic History.** Strong tradition of Swedish infrastructure history (Lennart Schön etc.). Possible dataset deposits.
- **Umeå universitet.** Northern Sweden / Inlandsbanan / Norrbotniabanan research.
- **Uppsala universitetsbibliotek (Carolina Rediviva).** Historical maps and railway company archives.
- **Kungliga biblioteket (KB).** National library — holds the digitized newspaper archive (*Svenska dagstidningar*). Every railway opening was front-page news. With OCR + named-entity-recognition + date extraction, you could build a date-of-opening dataset from newspaper coverage. Overkill for a personal project, but a fun ML angle if you want one.
- **DiVA portal.** Search for theses on "järnväg" + "GIS" / "historia" — student work sometimes ships with data appendices.
- **SND (Svensk nationell datatjänst).** Sweden's research data repository. Worth a search for railway-related datasets.

### 4.3 Museums & specialist organizations

- **Sveriges Järnvägsmuseum (Gävle).** Operated by Trafikverket. Research library with comprehensive archives. They have likely answered "when did segment X open" thousands of times — emailing their librarians is a reasonable shortcut for the last 5% of hard-to-date segments.
- **Svenska Järnvägsklubben (SJK).** Long-running enthusiast society that publishes detailed monographs ("Sveriges Järnvägar 150 år" and similar). Books-not-data, but exhaustive section-by-section opening dates.
- **Järnvägshistoriska Riksförbundet (JHRF).** Umbrella for ~70 local railway preservation societies. Each preserves a specific line and tends to have its own well-researched history.

### 4.4 Community & crowd-sourced

- **järnväg.net.** Community-maintained, *the* most comprehensive Swedish-language railway history site. Bandel-by-bandel (section-by-section) opening dates, closures, electrifications. Not formally a dataset — it's HTML pages — but well-structured and scrapable. Should be checked against terms of use first. Realistically the single most efficient web source for opening dates.
- **OpenStreetMap.** Geometry source for current and abandoned lines. Tags `start_date` / `end_date` exist but coverage is uneven.
- **OpenHistoricalMap (openhistoricalmap.org).** *(Niche but interesting.)* A fork of OSM specifically designed for historical features with start/end dates as first-class data. Swedish coverage is sparse, but the data model is *exactly* what we want — and contributing back our researched dataset would be a nice side benefit.
- **Wikidata SPARQL.** Query for all entities of type `Q728937` (railway line) located in Sweden, with `P571` (inception) and `P576` (dissolved/demolished date). Coverage varies by line, but many major lines are well-populated.
- **Wikipedia (sv).** Per-line articles with section tables. Structured similarly enough across articles that an LLM-assisted scrape is feasible.
- **DBpedia.** Sometimes has fields Wikidata is missing.

### 4.5 Creative angles worth considering

- **AI-assisted Wikipedia extraction.** Feed each `sv.wikipedia.org` line article through an LLM with a strict JSON schema ("for each section: start_station, end_station, length_km, opened_date, closed_date_or_null"). Results need spot-checking but get 90% coverage in an afternoon.
- **Newspaper OCR mining.** As mentioned under KB — overkill but cool.
- **Triangulate from station opening dates.** Wikidata has good coverage of *station* opening dates. A segment's opening year = `min(opening years of its endpoint stations)`. Rough but useful as a fallback.
- **Compare two years of Lantmäteriet historical maps** programmatically: vectorize railways from each, diff to find what was added/removed. Heavy ML work but in principle could date every segment to the nearest map series (~5–10 year resolution).
- **Just ask Sveriges Järnvägsmuseum.** A polite email explaining the project — museums often have spreadsheets internally that they'd happily share for a non-commercial visualization.

### 4.6 Recommended pipeline (revised)

1. **Spend half a day on data discovery first.** Specifically:
   - Email Martí-Henneberg's group at U. Lleida asking if their European railway GIS dataset covers Sweden adequately. If yes, skip steps 2–4.
   - Email Sveriges Järnvägsmuseum asking if they have or know of a section-level Swedish railway opening-date dataset.
   - Check SND and DiVA for existing depositions.
2. **Geometry:** Trafikverket NJDB (current heavy rail) + OSM (metro, light rail, abandoned). Auto-segment at junctions and station boundaries.
3. **Dates, in tiers:**
   - Tier 1: Wikidata SPARQL for the lines it covers well (~30–50% coverage expected).
   - Tier 2: LLM-assisted extraction from sv.wikipedia per-line articles (~40% more).
   - Tier 3: Manual fill-in from järnväg.net for the remainder.
   - Tier 4 (validation): cross-check yearly aggregate network length against SCB BiSOS Serie K tables. If our sum-of-segment-lengths-as-of-year-Y matches SCB's reported network length to within ~2%, the dataset is plausible. This is a really clean validation signal that I haven't seen used elsewhere.
4. **Ship-with-partial-data strategy:** start with the ~30 most historically significant lines (Tunnelbana, Inlandsbanan, Botniabanan, Citybanan, Citytunneln, Öresundsbron, Malmbanan, the major closed branch lines). A few hundred segments, tells most of the story. Expand from there.

---

## 5. Technical Architecture

### Stack
- **Single static HTML page** — no backend, no build step. Vanilla JS or a tiny ES module.
- **Google Maps JavaScript API** — `Map`, `Data` layer for GeoJSON, `InfoWindow`.
- **No framework** unless the slider/legend grows complex enough to justify one.

### File structure
```
index.html          # markup, slider, legend, map div, API key script tag
src/
  main.js           # map init, data load, slider wiring
  filter.js         # "is segment visible at year Y" + style function
  ui.js             # slider, play/pause, year panel, legend, category filters
data/
  segments.geojson
  operators.json
styles.css
scripts/
  fetch-trafikverket.js   # one-shot data import (run locally, output committed)
  fetch-osm.js            # Overpass API query for abandoned lines
  enrich-wikidata.js      # SPARQL → opening/closing years
```

### Rendering approach
- Load `segments.geojson` once into `map.data`.
- On year change: `map.data.setStyle(feature => styleFor(feature, currentYear, activeCategories))`.
- `styleFor` returns `{ visible, strokeColor, strokeWeight, zIndex }`. Hidden segments use `visible: false`.
- Newly-opened/closed-this-year segments get `strokeWeight: 6` + bright color for a brief flash.
- **Performance note:** with 2,000–5,000 polylines, `setStyle` on every slider tick is fine in modern browsers. If it stutters, switch to canvas-backed rendering via `OverlayView` + a single `<canvas>` redraw, but don't pre-optimize.

### Performance: file size
- 5,000 LineStrings as raw GeoJSON ≈ 5–15 MB. **Compress** the file by:
  - Reducing coordinate precision to 5 decimal places (~1 m).
  - Gzipping at the server (GitHub Pages does this automatically).
  - Optionally converting to TopoJSON for shared-boundary deduplication (saves ~30%).
- Target: <2 MB gzipped on the wire.

### Google Maps API key
- Restricted by HTTP referrer to the deploy domain.
- Loaded from `index.html` with `loading=async`.
- **Not committed to git** — `index.html` reads it from a `config.js` that is `.gitignore`d, with a `config.example.js` checked in.
- **Cost note:** Google Maps JS API has a free tier (~28,000 map loads/month). For a personal/portfolio project this is plenty. Make sure billing alerts are set up regardless.

### Hosting
- GitHub Pages or any static host. No server needed.

---

## 6. Out of Scope (v1)

- Mobile-optimized layout (desktop first).
- City trams (Gothenburg/Norrköping inner-city tram grids).
- Industrial sidings, museum railways, freight-only short spurs.
- Per-station markers with opening dates.
- Ridership, frequencies, electrification status visualization.
- Historical basemap tiles (today's Google Maps as basemap for all years).
- i18n — UI is English only. Place names stay in Swedish.

---

## 7. Open Questions for You

1. **Time range:** confirm 1950→today, *knowing* this is mostly a contraction story. Would you rather see 1850→today (the full growth-then-contraction arc) or 1856→today (since the first Swedish railway)?
2. **Closed lines:** include them (essential for showing the contraction) or only show what's currently in service (would make the visualization much less interesting given the 1950 cutoff)?
3. **City trams:** Gothenburg and Norrköping have extensive inner-city tram networks that have also expanded since 1950. Include or exclude?
4. **Coloring:** by category (recommended) or by operator?
5. **Data effort:** are you committing to full national coverage (a few weekends of manual data entry) or shipping with the ~30 most significant lines (a couple of evenings)?
6. **API key:** do you already have a Google Maps JS API key, or do we need to set one up?

---

## 8. Suggested Build Order

1. Static `index.html` with Google Maps showing Sweden. (sanity check on API key + zoom level)
2. Hardcoded sample of ~10 segments in `segments.geojson` covering one Tunnelbana line + one heavy rail line, rendered with hardcoded "year = 2024" filter.
3. Year slider wired up; visibility filter responds.
4. Play/pause auto-advance.
5. Category filter checkboxes + legend + year-info panel + tooltips.
6. Build the data pipeline (`scripts/`) and import the ~30 most significant lines.
7. "Newly opened/closed this year" highlight.
8. Expand dataset to full national coverage.
9. Polish, deploy.

Ship after step 5 with a small dataset to validate the UX before sinking time into national data entry.
