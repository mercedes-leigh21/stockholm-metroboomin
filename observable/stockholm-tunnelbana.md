---
title: Stockholm — A century in motion
theme: dark
---

# Stockholm — A century in motion

<small>Tunnelbana growth and city population, 1900 → 2024 · Data: Wikidata SPARQL · SCB API · Wikipedia</small>

```js
const stations  = await FileAttachment("stations.json").json();
const popData   = await FileAttachment("population.json").json();
const cityPop   = popData.city;
const countyPop = popData.county;
const COUNTY_FIRST_YEAR = countyPop[0].year;

// Catchment data: per-DeSO 2023 pop + first_served year (year of first station within 1km of centroid).
const catchment = await FileAttachment("catchment.json").json();
```

```js
const L = await import("https://cdn.jsdelivr.net/npm/leaflet@1.9.4/+esm").then(m => m.default);
```

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
<style>
  .leaflet-tile-pane { filter: brightness(0.32) saturate(0.35) contrast(1.05) hue-rotate(195deg); }
  .stat-card { background:#141b2e; border:1px solid rgba(255,255,255,0.07); border-radius:10px; padding:14px 16px; }
  .stat-card .label { font-size:9.5px; color:#5b6a85; text-transform:uppercase; letter-spacing:1.2px; font-weight:600; }
  .stat-card .value { font-size:26px; font-weight:700; margin-top:6px; font-variant-numeric:tabular-nums; }
  .stat-card.pop .value { color:#ffd989; }
  .stat-card.county .value { color:#c89060; }
  .stat-card.stn .value { color:#5cffe1; }
  .catchment-card { background: linear-gradient(160deg, rgba(255,217,137,0.10), rgba(255,217,137,0.02));
                    border: 1px solid rgba(255,217,137,0.25); border-radius:10px; padding:16px 18px; }
  .catchment-card .label { font-size:9.5px; color:#ffd989; font-weight:600; text-transform:uppercase; letter-spacing:1.3px; }
  .catchment-card .pct { font-size:52px; font-weight:800; line-height:1; margin:6px 0 4px; color:#ffd989; letter-spacing:-2px; font-variant-numeric:tabular-nums; }
  .catchment-card .pct sup { font-size:22px; font-weight:600; vertical-align:super; }
  .catchment-card .sub { font-size:12px; color:#8a9bb5; line-height:1.4; font-variant-numeric:tabular-nums; }
  .catchment-card .sub b { color:#f0f4fa; font-weight:600; }
</style>
```

```js
const lineColors = {
  "Röda linjen":  "#ff5252",
  "Blå linjen":   "#4cc9ff",
  "Gröna linjen": "#6ee06e"
};
const annotations = [
  { y: 1933, label: "Premetro tunnel" },
  { y: 1950, label: "First Tunnelbana opens" },
  { y: 1957, label: "T-Centralen unifies the system" },
  { y: 1960, label: "City pop peaks (808k)" },
  { y: 1975, label: "Blue line + ABC suburbs" },
  { y: 1981, label: "City pop bottoms (647k)" },
  { y: 2017, label: "Citybanan" }
];
```

```js
// Helpers
function interpAt(series, year) {
  if (year <= series[0].year) return series[0].pop;
  const last = series[series.length - 1];
  if (year >= last.year) return last.pop;
  for (let i = 1; i < series.length; i++) {
    if (series[i].year >= year) {
      const a = series[i - 1], b = series[i];
      const t = (year - a.year) / (b.year - a.year);
      return Math.round(a.pop + t * (b.pop - a.pop));
    }
  }
}
const popAt    = year => interpAt(cityPop, year);
const countyAt = year => year < COUNTY_FIRST_YEAR ? null : interpAt(countyPop, year);
const stationsAt = year => stations.filter(s => s.year <= year).length;
const fmt = d3.format(",");

// Pre-compute catchment curve once: year → cumulative pop_2023 in DeSOs served by then
const catchmentCurve = (() => {
  const sorted = catchment.deso.filter(r => r.first_served !== null)
                                .sort((a, b) => a.first_served - b.first_served);
  const m = new Map();
  let cum = 0, idx = 0;
  for (let y = 1900; y <= 2024; y++) {
    while (idx < sorted.length && sorted[idx].first_served <= y) { cum += sorted[idx].pop_2023; idx++; }
    m.set(y, cum);
  }
  return m;
})();
const catchmentTotal = catchment._total_pop_2023;
const catchmentAt = year => catchmentCurve.get(year) ?? 0;
const catchmentPctAt = year => (catchmentAt(year) / catchmentTotal) * 100;
```

```js
const year = view(Inputs.range([1900, 2024], { value: 1900, step: 1, label: "Year", width: "100%" }));
const showRings = view(Inputs.toggle({ label: "Show 1 km Tunnelbana catchment", value: false }));
```

<div class="grid grid-cols-2" style="gap: 14px;">
  <div class="card" style="padding:0; overflow:hidden;">
    ${tunnelbanaMap}
  </div>
  <div style="display:flex; flex-direction:column; gap:14px;">
    <div class="grid grid-cols-3" style="gap:10px;">
      <div class="stat-card pop">
        <div class="label">City</div>
        <div class="value" style="font-size:20px;">${fmt(popAt(year))}</div>
      </div>
      <div class="stat-card county">
        <div class="label">County</div>
        <div class="value" style="font-size:20px;">${countyAt(year) === null ? "—" : fmt(countyAt(year))}</div>
      </div>
      <div class="stat-card stn">
        <div class="label">T-bana</div>
        <div class="value" style="font-size:20px;">${stationsAt(year)}</div>
      </div>
    </div>
    <div class="catchment-card">
      <div class="label">Within 1 km of a Tunnelbana station</div>
      <div class="pct">${catchmentPctAt(year).toFixed(1)}<sup>%</sup></div>
      <div class="sub"><b>${fmt(catchmentAt(year))}</b> of today's <b>${fmt(catchmentTotal)}</b> Stockholmers</div>
    </div>
    <div class="card">${popChart}</div>
    <div class="card">${stnChart}</div>
  </div>
</div>

```js
// The map (created once, persists across year changes via mutable state on the DOM node)
const tunnelbanaMap = (() => {
  const div = html`<div style="height:520px; background:#050810;"></div>`;
  setTimeout(() => {
    const map = L.map(div, { center: [59.33, 18.0], zoom: 11 });
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "© OSM", maxZoom: 18
    }).addTo(map);
    const ghost = L.layerGroup().addTo(map);
    for (const s of stations) {
      L.circleMarker([s.lat, s.lon], {
        radius: 2.5, color: '#ffffff', opacity: 0.18,
        fillColor: '#ffffff', fillOpacity: 0.15, weight: 1, interactive: false
      }).addTo(ghost);
    }
    div._map = map;
    div._activeLayer = L.layerGroup().addTo(map);
  }, 0);
  return div;
})();
```

```js
// Reactive: redraw markers when year changes
{
  const map = tunnelbanaMap._map;
  if (!map) return;
  const layer = tunnelbanaMap._activeLayer;
  layer.clearLayers();
  const RECENT = 3;
  for (const s of stations) {
    if (s.year > year) continue;
    const isNew = s.year > year - RECENT;
    const isInterchange = s.lines.length >= 2;
    const baseColor = lineColors[s.lines[0]] || '#888';
    L.circleMarker([s.lat, s.lon], {
      radius: isNew ? 9 : (isInterchange ? 7 : 6),
      color: isNew ? '#ffffff' : (isInterchange ? '#ffd989' : '#ffffff'),
      weight: 2,
      fillColor: isNew ? '#ffe45c' : baseColor,
      fillOpacity: 1.0
    }).bindTooltip(
      `<b>${s.name}</b><br>${s.lines.join(' / ')}<br>Opened ${s.year}`,
      { direction: 'top', offset: [0, -6] }
    ).addTo(layer);
  }
}
```

```js
// Reactive: 1 km catchment rings around each open station (year-aware, toggleable)
{
  const map = tunnelbanaMap._map;
  if (!map) return;
  if (tunnelbanaMap._rings) { map.removeLayer(tunnelbanaMap._rings); tunnelbanaMap._rings = null; }
  if (!showRings) return;
  const layer = L.layerGroup();
  for (const s of stations.filter(s => s.year <= year)) {
    L.circle([s.lat, s.lon], {
      radius: 1000,
      color: "rgba(255, 217, 137, 0.4)", weight: 0.5,
      fillColor: "rgba(255, 217, 137, 1)", fillOpacity: 0.10,
      interactive: false
    }).addTo(layer);
  }
  layer.addTo(map);
  tunnelbanaMap._rings = layer;
  invalidation.then(() => { if (tunnelbanaMap._rings === layer) { map.removeLayer(layer); tunnelbanaMap._rings = null; } });
}
```

```js
// Suburbs (county minus city) stack: each row has y1=city pop (bottom of band) and y2=county pop (top).
// Only valid for years 1968+; for earlier years the city layer renders alone.
const countyStack = countyPop.map(d => ({ year: d.year, y1: popAt(d.year), y2: d.pop }));
const countyClipPast   = countyStack.filter(d => d.year <= year).concat(
  year >= COUNTY_FIRST_YEAR ? [{year, y1: popAt(year), y2: countyAt(year)}] : []
);
const countyClipFuture = countyStack.filter(d => d.year >= year);
const popTopAtPlayhead = countyAt(year) ?? popAt(year);
```

```js
const popChart = Plot.plot({
  height: 200, width: 440,
  marginLeft: 50, marginTop: 18, marginBottom: 28,
  style: { background: "transparent", color: "#8a9bb5", fontSize: 10 },
  x: { domain: [1900, 2024], label: null, tickFormat: "d", tickSize: 4 },
  y: { label: "Greater Stockholm population", tickFormat: d => d/1000 + "k", grid: true, nice: true },
  marks: [
    Plot.ruleX(annotations.map(a => a.y), { stroke: "#5b6a85", strokeOpacity: 0.3, strokeDasharray: "2,3" }),

    // City layer (bottom)
    Plot.areaY(cityPop.filter(d => d.year >= year),
      { x: "year", y: "pop", curve: "monotone-x", fill: "#ffd989", fillOpacity: 0.08 }),
    Plot.lineY(cityPop.filter(d => d.year >= year),
      { x: "year", y: "pop", curve: "monotone-x", stroke: "#ffd989", strokeOpacity: 0.3, strokeWidth: 1.5 }),
    Plot.areaY(cityPop.filter(d => d.year <= year).concat([{year, pop: popAt(year)}]),
      { x: "year", y: "pop", curve: "monotone-x", fill: "#ffd989", fillOpacity: 0.35 }),
    Plot.lineY(cityPop.filter(d => d.year <= year).concat([{year, pop: popAt(year)}]),
      { x: "year", y: "pop", curve: "monotone-x", stroke: "#ffd989", strokeWidth: 2 }),

    // Suburbs layer (county minus city), stacked above city
    Plot.area(countyClipFuture,
      { x: "year", y1: "y1", y2: "y2", curve: "monotone-x", fill: "#c89060", fillOpacity: 0.08 }),
    Plot.line(countyClipFuture,
      { x: "year", y: "y2", curve: "monotone-x", stroke: "#c89060", strokeOpacity: 0.3, strokeWidth: 1.5 }),
    Plot.area(countyClipPast,
      { x: "year", y1: "y1", y2: "y2", curve: "monotone-x", fill: "#c89060", fillOpacity: 0.35 }),
    Plot.line(countyClipPast,
      { x: "year", y: "y2", curve: "monotone-x", stroke: "#c89060", strokeWidth: 2 }),

    // Playhead
    Plot.ruleX([year], { stroke: "white", strokeOpacity: 0.5, strokeDasharray: "4,4" }),
    Plot.dot([{x: year, y: popTopAtPlayhead}], { x: "x", y: "y", fill: "#ffe45c", stroke: "white", strokeWidth: 1.5, r: 4 })
  ]
});
```

```js
const stnSeries = d3.range(1900, 2025).map(y => ({ year: y, n: stationsAt(y) }));
```

```js
const stnChart = Plot.plot({
  height: 180, width: 440,
  marginLeft: 48, marginTop: 18, marginBottom: 28,
  style: { background: "transparent", color: "#8a9bb5", fontSize: 10 },
  x: { domain: [1900, 2024], label: null, tickFormat: "d", tickSize: 4 },
  y: { label: "Cumulative stations", labelArrow: false, grid: true, nice: true },
  marks: [
    Plot.ruleX(annotations.map(a => a.y), { stroke: "#5b6a85", strokeOpacity: 0.3, strokeDasharray: "2,3" }),
    Plot.areaY(stnSeries.filter(d => d.year >= year), { x: "year", y: "n", curve: "step-after", fill: "#5cffe1", fillOpacity: 0.08 }),
    Plot.lineY(stnSeries.filter(d => d.year >= year), { x: "year", y: "n", curve: "step-after", stroke: "#5cffe1", strokeOpacity: 0.3, strokeWidth: 1.5 }),
    Plot.areaY(stnSeries.filter(d => d.year <= year), { x: "year", y: "n", curve: "step-after", fill: "#5cffe1", fillOpacity: 0.35 }),
    Plot.lineY(stnSeries.filter(d => d.year <= year), { x: "year", y: "n", curve: "step-after", stroke: "#5cffe1", strokeWidth: 2 }),
    Plot.ruleX([year], { stroke: "white", strokeOpacity: 0.5, strokeDasharray: "4,4" }),
    Plot.dot([{x: year, y: stationsAt(year)}], { x: "x", y: "y", fill: "#ffe45c", stroke: "white", strokeWidth: 1.5, r: 4 })
  ]
});
```

---

The Tunnelbana opened in 1950 right as Stockholm city hit its all-time peak (808k in 1960). Over the next 20 years the city *shrank* as the metro enabled the ABC-städer suburbs (Vällingby, Farsta, Skärholmen). The city core only recovered after 2000.

*Sources: [Wikidata SPARQL](https://www.wikidata.org/wiki/Q272926) for stations · [SCB API](https://api.scb.se) for population 1968–2024 · Wikipedia for population 1900–1960.*
