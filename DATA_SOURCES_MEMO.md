# Data Sources Verification Memo

Findings from a one-pass verification on each candidate source listed in [SPEC.md §4](SPEC.md). All numbers measured live, not estimated.

---

## TL;DR

- **Geometry: solid.** OSM has 33,185 railway ways for Sweden, fully downloadable. Trafikverket NJDB available via Lastkajen with free registration.
- **Dates: harder than I claimed.** Wikidata only has inception dates for ~10% of Swedish railway lines. OSM has `start_date` on ~11% of railway ways. So the "easy" structured-data sources cover ~10% — not enough.
- **Best primary source for dates is Project Runeberg.** SJ's official 50-year (1856–1906) and 75-year (1906–1931) commemorative histories are fully digitized, OCR-searchable, free. These are authoritative section-by-section opening records.
- **Martí-Henneberg European dataset exists but is not openly downloadable.** Confirmed in academic publications. No public download URL found. Realistically: email Prof. Martí-Henneberg or Max Satchell at Cambridge.
- **järnväg.net and historiskt.nu are the practical answer for the modern era** but both lack stated licensing — would need permission for republication.

**Recommended path:** OSM (geometry) + Wikidata (dates where available) + Project Runeberg LLM-extraction for pre-1931 + manual fill-in from järnväg.net for 1931–today, validated against SCB BiSOS L aggregate network length.

---

## Source-by-source findings

### 1. SCB — BiSOS Serie L "Statens järnvägstrafik" (1862–1910)

✅ **Verified.** Series exists, covers 1862–1910.

- **Correction to the spec:** I had the series letter wrong. BiSOS K is healthcare; **BiSOS L is railways.**
- Continuation series exists: SM Serie D "Järnvägsstatistiska meddelanden" covers 1912–1953.
- Contains: maps showing network growth in nearly every issue (so per-year geometry snapshots exist), annual freight traffic supplements from 1904, ore-railway/locomotive/carriage/ferry appendices 1909–1910.
- **Format:** SCB page mentions "digitaliserade publikationer" but doesn't give direct PDF links. Likely via Project Runeberg.
- **Best use:** **validation signal**, not primary data — yearly network length tables let us check that our segment dataset sums to the right total.
- Source: [scb.se BiSOS L](https://www.scb.se/hitta-statistik/aldre-statistik/innehall/bidrag-till-sveriges-officiella-statistik/statens-jarnvagstrafik-1862-1910-bisos-l/)

### 2. Trafikverket — NJDB via Lastkajen

✅ **Verified accessible.**

- Public download portal at [trafikverket.se/e-tjanster/lastkajen](https://www.trafikverket.se/e-tjanster/lastkajen--sveriges-vag--och-jarnvagsdata/).
- Requires free registration + license acceptance. Pre-built data packages or custom orders.
- Covers all currently-operational rail (Trafikverket + Inlandsbanan + Arlandabanan + Öresundsbron + private actors).
- Web browsing service at [njdbwebb.trafikverket.se](https://njdbwebb.trafikverket.se/).
- **No historical layers** — current network only.
- **Best use:** authoritative current heavy-rail geometry, segmented by official "bandel" boundaries that match how Swedish railway history is conventionally documented.

### 3. Martí-Henneberg European railway GIS dataset

⚠️ **Exists but not openly downloadable.**

- Confirmed in [ScienceDirect publication, 2013](https://www.sciencedirect.com/science/article/abs/pii/S0966692312002517) and [open-access SciRP article](https://www.scirp.org/html/13-8401140_18790.htm). Pan-European, 10-year intervals, 1840–2010.
- Cambridge campop page only contains the Britain-only subset (under Max Satchell's authorship), not the full European data.
- **Conclusion:** would need an email to Prof. Jordi Martí-Henneberg (Universitat de Lleida) or Dr. Max Satchell (Cambridge) to request access. **Action you'd need to take, not me.**

### 4. Wikidata SPARQL — Swedish railway lines

✅ **Tested live.** Numbers are real.

| Metric | Count |
|---|---|
| Swedish railway-line entities (P31/P279* `Q728937`, P17 `Q34`) | **675** |
| ...with inception date (P571) | **65** (~9.6%) |
| ...with both inception and dissolution dates | many of the 65 |

- The 65 with dates look high quality — pre-1900 narrow-gauge lines, dissolution dates for closed companies, well-labeled in Swedish.
- **But 9.6% coverage means Wikidata cannot be the primary date source.**
- **Best use:** seed dataset for the 65 lines it covers well, plus opportunity to *contribute back* opening dates we research from other sources.
- Stations might have better coverage; didn't measure separately. Worth a follow-up query if we go this route.

### 5. järnväg.net

⚠️ **Best practical source for opening dates, but no download and no license.**

- Per-section ("bandel") fact boxes contain: opening year, length (km), gauge (mm), electrification, max speed, axle load, double/quad-track sections, signaling.
- Closed lines documented separately as "nedlagda banor."
- **Format:** HTML pages organized by region only. No API, no dataset.
- **Licensing:** none stated. Republishing scraped data would require contacting the maintainers (Trafikverket's Karl Rosander appears to be involved).
- **Best use:** scrape *for our own reference* during data entry, but don't redistribute scraped content as-is. Use it to look up dates that we then store in our own dataset.

### 6. historiskt.nu — bandelsregister

✅ **Comprehensive, also browse-only, also no license.**

- Has bandelar by number, station name, and gauge; operating facilities through 2009; railway companies with signatures and gauges.
- Sources cited: 1921 "Sveriges Järnvägar," 1953 "Svensk Järnvägs- och stationsförteckning."
- Same constraints as järnväg.net: browse-only, no licensing statement.
- Maintained by Kjell Byström and Rolf Sten — known names in Swedish railway history.

### 7. OpenStreetMap — current and abandoned railways

✅ **Tested live.**

| Query (Sweden, all railway types) | Count |
|---|---|
| All railway ways (`rail`/`subway`/`light_rail`/`tram`/`narrow_gauge`/`abandoned`/`disused`/`construction`) | **33,185** |
| ...with `start_date` tag | **3,522** (~10.6%) |

- **Geometry is essentially complete.** Coverage of abandoned lines surprisingly good given community contributions.
- **Date coverage matches Wikidata's ~10%** — nowhere near enough on its own.
- License: ODbL — usable with attribution.

### 8. OpenHistoricalMap

⚠️ **Ideal data model, sparse Swedish coverage.**

- **Tested live via OHM's Overpass API:** ~7,254 railway ways within the Sweden bounding box.
- That's ~22% of OSM's count for the same area. Many of the OHM features may be the same modern lines copied over without proper start_date enrichment — coverage of *historical* railways specifically not measured (would need a `start_date<1950` filter).
- Data model is exactly right (start_date / end_date as first-class).
- **Best use:** *contribute back* — once we build our dataset, upload it to OHM as a community contribution. But not a starting source.

### 9. Project Runeberg — primary historical sources

✅ **Strongest finding of the verification pass.**

- **"Statens järnvägar 1856-1906"** ([runeberg.org/sj50/](https://runeberg.org/sj50/)): 4-part 50-year history published 1906 by the railway administration itself. Covers history, infrastructure & buildings, transport equipment, administration. Full text, OCR-searchable, downloadable.
- **"Statens järnvägar 1906-1931"** ([runeberg.org/sj1931/](https://runeberg.org/sj1931/)): 75-year continuation, similarly comprehensive.
- These are the **authoritative primary source** for line-by-line opening dates up to 1931, written by the railway operator themselves.
- Combined with järnväg.net for 1931→today, this covers the full timeline.
- Free, public domain, no licensing constraints.

### 10. EU GISCO / European Data Portal rail data

⚠️ **Likely current-only, not historical.** Couldn't verify content of the specific dataset (page rendered as a stub). Common European rail GIS layers are operational-network snapshots, not time-series. Would deprioritize.

---

## Revised data pipeline recommendation

Based on what's actually verified to exist:

**Geometry (current network):**
- OSM via Overpass API for everything (heavy rail + metro + light rail + abandoned). ODbL.
- *Optionally* cross-check operational heavy rail against Trafikverket NJDB (requires registration).

**Opening/closing dates, in priority order:**
1. **Wikidata SPARQL** — 65 lines for free, structured. Use as-is.
2. **LLM-extracted Project Runeberg SJ histories** — covers 1856–1931 systematically from authoritative primary sources. This is the biggest unlock and didn't exist in my original spec.
3. **LLM-extracted sv.wikipedia line articles** — covers most named lines with section tables.
4. **Manual lookup in järnväg.net / historiskt.nu** for stragglers — store dates in our dataset, don't republish their content.
5. **(Out of reach without email)** Martí-Henneberg European dataset — would short-circuit much of this if accessible.

**Validation:**
- For each year 1862–1910, sum our dataset's segment lengths and compare against SCB BiSOS L published network length. Discrepancies > ~2% point to bugs.
- For 1912–1953, validate against SM Serie D.
- For 1953–today, Trafikanalys / Trafikverket aggregate stats.

---

## What to update in SPEC.md

1. Fix series letter: **BiSOS L**, not K (K is healthcare).
2. Add Project Runeberg SJ histories as the primary pre-1931 date source.
3. Downgrade Wikidata expectation from "30–50% coverage" to "~10%".
4. Downgrade OSM `start_date` expectation from "20–40%" to "~11%".
5. Add the SCB validation idea more prominently — it's a genuinely novel quality check.
6. Note that Martí-Henneberg requires an email, not a download.

---

## Open questions you'd need to answer before we build

1. **Email outreach:** are you willing to email Martí-Henneberg's group? It's a low-effort, high-upside ask.
2. **järnväg.net licensing:** are you willing to email them too, asking permission to republish opening dates we cite from their site?
3. **Time investment:** scraping + LLM-extracting Project Runeberg's two SJ histories is realistically 1–2 days of careful work to get a clean dataset. That's the load-bearing piece. Are you committing to it?
