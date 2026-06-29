# fid-analytical-dashboard — Complete Walkthrough

A line-by-line explanation of the Nuxeo analytical dashboard: how the two files
connect, what every HTML slot does, the CSS behind each piece, every helper in
the script, and how the whole thing fits together.

This dashboard is a **Polymer 2 Web Component** for Nuxeo Web UI (LTS 2023). It
tracks document growth, storage growth, file-type/format breakdowns, and
migration adoption — and it is designed to work at **billion-document scale**.

---

## Table of Contents

1. [The two-file setup (how they connect)](#1-the-two-file-setup-how-they-connect)
2. [The data elements](#2-the-data-elements-top-of-the-template)
3. [The page shell](#3-the-page-shell)
4. [The header card (title + period dropdown)](#4-the-header-card-title--period-dropdown)
5. [The KPI strip](#5-the-kpi-strip)
6. [Slot 1 — the hero (tabs + charts)](#6-slot-1--the-hero-tabs--charts)
7. [Slot 2 — Migration Adoption (the donut)](#7-slot-2--migration-adoption-the-donut)
8. [Slots 3–6 — placeholders](#8-slots-36--placeholders)
9. [Responsive section](#9-responsive-section-bottom-of-the-css)
10. [The script — every helper explained](#10-the-script--every-helper-explained)
11. [The script as a whole — the big picture](#11-the-script-as-a-whole--the-big-picture)

---

## 1. The two-file setup (how they connect)

At the very top of the main file:

```html
<link rel="import" href="fid-dashboard-styles.html">
```

This pulls in the styles file. Then inside the template:

```html
<style include="iron-flex iron-flex-alignment nuxeo-styles fid-dashboard-styles"></style>
```

`include=` is Polymer's way of importing **style modules** by their `dom-module`
id. Four are pulled in:

- `iron-flex` and `iron-flex-alignment` — Nuxeo's flexbox helper classes
- `nuxeo-styles` — the global Nuxeo theme variables like `--nuxeo-text-default`
- `fid-dashboard-styles` — your own CSS module

So all the CSS lives in the second file, and the main file just references it by
name. **That is why both files must be in the bundle** — the main file is useless
without the style module it names.

In the **styles file**, everything is wrapped as:

```html
<dom-module id="fid-dashboard-styles">
  <template><style> /* ...all your CSS... */ </style></template>
</dom-module>
```

The `id` here is exactly the name used in the `include=` above. That `id` is the
whole link between the two files.

---

## 2. The data elements (top of the template)

These have no appearance — they fetch numbers. They sit at the top so they load
as the element initializes.

### Aggregation queries (charts)

```html
<nuxeo-repository-data
  start-date="[[startDate]]"
  end-date="[[_extendEndDate(endDate)]]"
  with-date-intervals="month"
  date-field="dc:created"
  data="{{docsPerMonth}}"
  index="[[index]]">
</nuxeo-repository-data>
```

- `[[startDate]]` is **one-way binding** (data flows in).
- `{{docsPerMonth}}` is **two-way** (the element writes its result back out into
  your `docsPerMonth` property).

When `startDate`/`endDate` change, this re-runs automatically and refills
`docsPerMonth`. There are five such aggregations: docs per month, storage per
month, by doc type, by format, and total storage.

### Count queries (migration) — a different element, for scale

```html
<nuxeo-page-provider
  auto
  page-size="1"
  query="SELECT * FROM Document WHERE ... dc:created BETWEEN ? AND ?"
  query-params="[[_countParams(startDate, endDate)]]"
  number-of-results="{{totalCount}}">
</nuxeo-page-provider>
```

`page-size="1"` + reading only `number-of-results` means it **counts matches
without downloading documents** — that is the billion-scale trick. The `? AND ?`
placeholders are filled by `_countParams`, which returns `[startDate, endDate]`.

No CSS is involved yet — these elements are invisible.

---

## 3. The page shell

```html
<div class="page">
  <div class="content">
    <nuxeo-page>
      <div slot="header"><span class="flex">Analytical Dashboard</span></div>
      <div class="dashboard-container"> <!-- everything --> </div>
```

**CSS for `.page` and `.content`:**

```css
.page, .content { @apply --layout-vertical; }
```

`@apply --layout-vertical` is a Polymer mixin (from iron-flex) meaning "stack
children vertically" — equivalent to `display:flex; flex-direction:column`.

**CSS for `.dashboard-container`:**

```css
.dashboard-container {
  padding: 16px 24px 32px 24px;
  background: var(--nuxeo-page-background, #f0f2f5);
  min-height: 100vh;
  box-sizing: border-box;
}
```

This is the grey backdrop everything sits on.

- `var(--nuxeo-page-background, #f0f2f5)` means "use Nuxeo's theme background; if
  undefined, fall back to `#f0f2f5`."
- `min-height: 100vh` makes it fill the screen height.
- `box-sizing: border-box` makes padding count *inside* the width so nothing
  overflows.

`<div slot="header">` places that title into Nuxeo's dark admin bar —
`class="flex"` (from iron-flex) makes it expand to fill space.

---

## 4. The header card (title + period dropdown)

```html
<nuxeo-card class="dates">
  <div class="header-slot">
    <div class="header-left"> <h1>...</h1> <p>...</p> </div>
    <div class="header-divider"></div>
    <div class="header-controls"> <!-- dropdown + custom dates --> </div>
  </div>
</nuxeo-card>
```

**`.header-slot`** — the row that holds everything:

```css
.header-slot {
  display: flex; flex-direction: row;
  align-items: center; justify-content: space-between;
  flex-wrap: nowrap; gap: 24px; padding: 8px 4px;
}
```

`justify-content: space-between` is what pushes the **title to the far left and
controls to the far right**. `align-items: center` vertically centres them.
`gap: 24px` spaces the three children.

**`.header-left`** — the title block:

```css
.header-left { display:flex; flex-direction:column; gap:4px; flex:1; min-width:0; }
```

`flex:1` lets it grow to absorb free space (pushing controls right).
`min-width:0` is a flexbox quirk fix — it lets the text truncate instead of
overflowing.

**`.header-left h1` / `p`:** font sizes and colours from theme variables. The
trio `white-space:nowrap; overflow:hidden; text-overflow:ellipsis` means a long
title becomes "Analytical Dash…" rather than wrapping.

**`.header-divider`** — the thin vertical line between title and controls:

```css
.header-divider { width:1px; height:40px; background: var(--nuxeo-border,#e0e0e0); flex-shrink:0; }
```

`flex-shrink:0` stops it from being squished to nothing.

**`.header-controls`:**

```css
.header-controls { display:flex; flex-direction:row; align-items:flex-end; gap:16px; flex-shrink:0; flex-wrap:wrap; }
```

Holds the dropdown and date pickers in a row, bottom-aligned
(`align-items:flex-end`) so labels and inputs line up at their base.

The dropdown itself:

```html
<nuxeo-select selected="{{selectedPeriod}}" options='[...]' on-selected-changed="_onPeriodChanged">
```

`{{selectedPeriod}}` two-way binds the choice; `on-selected-changed` also fires
`_onPeriodChanged`.

The custom date inputs only appear when needed:

```html
<template is="dom-if" if="[[_isCustom(selectedPeriod)]]">
```

`dom-if` stamps its contents into the page **only when** `_isCustom` returns true
(period === 'custom').

**`.custom-dates`, `.date-field`, `.date-field label`, `.date-field input`** —
these style the two date pickers: stacked label-over-input columns, with the
input getting a border that turns blue on focus (`:focus` rule with `box-shadow`
glow).

---

## 5. The KPI strip

```html
<div class="flex-layout tight">
  <nuxeo-card class="full-width">
    <div class="kpi-strip">
      <div class="kpi-cell"><span class="kpi-value">...</span><span class="kpi-label">...</span></div>
      <!-- four cells -->
```

**`.flex-layout`** — the generic card-grid container, reused everywhere:

```css
.flex-layout { display:flex; flex-wrap:wrap; margin:16px -8px 0 -8px; }
.flex-layout.tight { margin-top:8px; }
```

The negative side margins (`-8px`) cancel out the `8px` margins on the cards
inside, so cards align flush to the edges. `.tight` just reduces the top gap for
the KPI strip sitting right under the header.

**`.flex-layout nuxeo-card`** — how every card sizes:

```css
.flex-layout nuxeo-card { flex: 1 0 calc(33% - 16px); margin: 0 8px 16px 8px; }
.flex-layout nuxeo-card.full-width { flex: 1 0 calc(100% - 16px); }
```

`flex: 1 0 calc(33% - 16px)` = "each card takes about a third of the row" (the
`-16px` accounts for margins). Adding `full-width` makes it span the whole row.
**This one rule is what controls the 3-per-row grid.** The KPI card and the hero
both use `full-width`; the migration and placeholder cards don't, so they fall
into thirds.

**`.kpi-strip` and `.kpi-cell`:**

```css
.kpi-strip { display:flex; flex-direction:row; flex-wrap:wrap; align-items:stretch; }
.kpi-cell { flex:1 1 0; min-width:120px; padding:4px 20px; border-right:1px solid var(--nuxeo-border,#eee); }
.kpi-cell:last-child { border-right:none; }
```

The cells sit in a row, each taking equal width (`flex:1 1 0`), separated by thin
right borders — except the last (`:last-child` removes its border). `.kpi-value`
is the big number, `.kpi-label` the small grey caption.

The bindings inside:

```html
<span class="kpi-value">[[_formatCount(totalCount)]]</span>            <!-- "296" or "1.2M" -->
<span class="kpi-value">[[_formatStorage(totalStorageBytes)]]</span>  <!-- "460.7 KB" -->
<span class="kpi-value">[[migratedCount]]</span>                      <!-- exact count -->
<span class="kpi-value">[[_nonMigrated(totalCount, migratedCount)]]</span>
```

---

## 6. Slot 1 — the hero (tabs + charts)

```html
<nuxeo-card class="full-width" heading="Growth Overview">
  <div class="view-tabs">
    <span class$="[[_tabClass(activeView,'overall')]]" on-tap="_selectOverall">Overall</span>
    <!-- three more tabs -->
  </div>
  <template is="dom-if" if="[[_isView(activeView,'overall')]]"> <!-- chart --> </template>
  <!-- three more dom-if blocks -->
</nuxeo-card>
```

The tabs use `class$=` (note the `$`). That is Polymer's syntax for binding a
**whole attribute value** — here the class string. `_tabClass` returns either
`"view-tab"` or `"view-tab active"`. `on-tap` switches which view is active. Each
chart sits in a `dom-if` that only renders when its tab is active — so only one
chart exists in the DOM at a time.

**`.view-tabs` and `.view-tab`:**

```css
.view-tabs { display:flex; flex-direction:row; gap:4px; margin:4px 0 12px 0; flex-wrap:wrap; }
.view-tab { font-size:12px; font-weight:500; color:#777; padding:6px 14px; border-radius:16px; cursor:pointer; user-select:none; transition: background .15s, color .15s; }
.view-tab:hover { background: var(--nuxeo-page-background,#f0f2f5); }
.view-tab.active { background: var(--nuxeo-primary-color,#1a73e8); color:white; }
```

The pills. `border-radius:16px` rounds them. `.active` fills the selected one
blue with white text. `transition` makes the colour change smooth.
`cursor:pointer` shows a hand; `user-select:none` stops text from being selected
on click.

**`.chart-sub-title`** — the small label above each chart (e.g. "Documents and
storage added per month"): just font/colour/margin.

**`.chart-box`** — the critical one for chart sizing:

```css
.chart-box { position:relative; width:100%; height:380px; margin-top:8px; }
chart-line, chart-bar, chart-pie { display:block; width:100% !important; height:100% !important; }
```

Chart.js needs a **fixed-height parent** to draw into (with
`maintainAspectRatio:false` in the options). The `380px` height here is what
gives the chart room; the chart then fills it 100%. Without this fixed height,
the chart would collapse or overlap — the exact bug seen earlier in development.

**`.chart-empty`** — the "No documents found" message shown when a pie's data is
empty (via the `_isEmpty` dom-if).

The Overall chart is special — it passes an **options object** built by a helper
rather than inline JSON:

```html
<chart-bar values="[[_overallValues(...)]]" series="[[_overallSeries(...)]]" options="[[_overallOptions(...)]]">
```

because it needs storage as bars + documents as a line on two Y-axes, which is
too complex for an inline string.

---

## 7. Slot 2 — Migration Adoption (the donut)

```html
<nuxeo-card heading="Migration Adoption">     <!-- no full-width, so it's one-third -->
  <div class="donut-wrap">
    <chart-pie ... options='{... "cutoutPercentage":68 ...}'></chart-pie>
    <div class="donut-center"><span class="pct">[[_migratedPct(...)]]%</span><span class="pct-label">MIGRATED</span></div>
  </div>
  <div class="migration-summary"> <!-- three stats --> </div>
</nuxeo-card>
```

`cutoutPercentage:68` is what turns the pie into a **donut** (68% of the centre
cut out). The hole is then filled by an overlay.

**`.donut-wrap` and `.donut-center`:**

```css
.donut-wrap { position:relative; width:100%; height:240px; }
.donut-center { position:absolute; top:0; left:0; right:0; bottom:0; display:flex; flex-direction:column; align-items:center; justify-content:center; pointer-events:none; }
```

The wrap is `position:relative` so the center can be `position:absolute`
**inside it**, stretched to all four edges (`top/left/right/bottom:0`) and
flex-centered — that is how the "62% MIGRATED" text floats dead-center in the
donut hole. `pointer-events:none` lets mouse hovers pass through to the chart
beneath (so tooltips still work).

**`.donut-center .pct`** (big green %) and **`.pct-label`** (small grey
"MIGRATED").

**`.migration-summary` and `.stat`:**

```css
.migration-summary { display:flex; flex-direction:row; justify-content:center; gap:24px; margin-top:12px; }
.migration-summary .stat { display:flex; flex-direction:column; align-items:center; }
```

The three little number/label columns (Migrated / Non-Migrated / Total) centered
below the donut.

---

## 8. Slots 3–6 — placeholders

```html
<nuxeo-card heading="Slot 3">
  <div class="slot-placeholder">
    <iron-icon icon="icons:insert-chart"></iron-icon>
    <span class="placeholder-label">Coming soon</span>
  </div>
</nuxeo-card>
```

**`.slot-placeholder`:**

```css
.slot-placeholder { display:flex; flex-direction:column; align-items:center; justify-content:center; min-height:200px; gap:10px; }
.slot-placeholder iron-icon { --iron-icon-width:40px; --iron-icon-height:40px; color: var(--nuxeo-border,#ddd); }
.slot-placeholder .placeholder-label { font-size:12px; color:#bbb; }
```

Centers a faded chart icon over "Coming soon". `min-height:200px` keeps these
cards roughly the same height as the real ones. These cards have no `full-width`,
so they flow as thirds alongside the migration card.

---

## 9. Responsive section (bottom of the CSS)

```css
@media (max-width: 1100px) { .flex-layout nuxeo-card { flex: 1 0 calc(50% - 16px); } }
@media (max-width: 768px)  { .flex-layout nuxeo-card { flex: 1 0 calc(100% - 16px); } /* header stacks */ }
```

Two breakpoints:

- **Under 1100px** wide, cards become **half-width** (2 per row).
- **Under 768px** (phones), cards go **full-width** (1 per row), the header
  switches from a row to a stack, the divider hides, and the dropdown/date inputs
  go full-width.

This is the entire responsive behavior — driven by changing that one `flex`
basis.

---

## 10. The script — every helper explained

### Lifecycle

- **`properties`** — declares every variable. `selectedPeriod` has
  `observer:'_onPeriodChanged'` (runs that function whenever it changes) and a
  default of `'6months'`. The data properties default to `[]` or `0` so the
  charts have something safe to render before data arrives.
- **`ready()`** — runs once when the element loads; calls `_computeDates` so the
  dashboard opens with a populated 6-month range.

### Period helpers

| Helper | What it does |
|--------|--------------|
| `_isCustom(period)` | True if period is `'custom'`; drives the date-picker `dom-if`. |
| `_onPeriodChanged(newVal)` | Runs on dropdown change; ignores `'custom'` (handled by manual pickers), else computes dates. |
| `_computeDates(period)` | The core: looks up how many months the period means (`monthMap`), sets `start = today − N months`, `end = today`, stores both as ISO strings. Setting these triggers every data element to refetch. |
| `_onCustomDateChanged()` | When both custom inputs are filled, copies them into `startDate`/`endDate`. |
| `_toISODate(date)` | Turns a JS Date into `"YYYY-MM-DD"`. |
| `_extendEndDate(date)` | Pushes the end to the last millisecond of the day, so "today's" documents aren't missed by the range. |
| `_countParams(startDate, endDate)` | Packs `[start, end]` for the page-provider's `? AND ?`; returns `null` until both exist so the count queries don't fire half-formed. |

### Tab helpers

| Helper | What it does |
|--------|--------------|
| `_isView(active, view)` | True if a given tab is the active one; controls which chart `dom-if` renders. |
| `_tabClass(active, view)` | Returns `"view-tab active"` or `"view-tab"` for the pill styling. |
| `_selectOverall` / `_selectDocType` / `_selectFormat` / `_selectSize` | Set `activeView` on click. |

### Generic chart helpers

ES returns buckets like `[{ key: "...", value: N }, ...]`.

| Helper | What it does |
|--------|--------------|
| `_isEmpty(data)` | True if an aggregation returned nothing (drives empty-state messages). |
| `_labels(data)` | Pulls the `key` of each bucket for axis/pie labels. |
| `_values(data)` | Pulls the `value` of each bucket, wrapped as `[[...]]` because chart-elements expects an array of series. |
| `_monthLabels(data)` / `_toMonthYear(key)` | Convert `"2026-03-01"` into a friendly `"Mar-26"` for the x-axis. |

### Overall combined-chart helpers

| Helper | What it does |
|--------|--------------|
| `_docsAlignedToStorage(storage, docs)` | Lines up the document counts to the same months as storage, filling 0 for any missing month, so both datasets share an x-axis. |
| `_overallValues(storage, docs)` | Returns two series: storage (converted to the chosen unit) and documents. |
| `_overallSeries(storage)` | The two legend labels, e.g. `["Storage (MB)", "Documents"]`. |
| `_overallOptions(storage)` | Builds the Chart.js config object: tells it dataset-2 is a line on a right-hand axis, sets the two Y-axes, gridlines, tooltips. Returned as an object (not a string) because of its complexity. |

### Storage helpers (auto-scaling units)

| Helper | What it does |
|--------|--------------|
| `_storageMax(data)` | Finds the biggest monthly byte value. |
| `_storageUnit(data)` | Picks KB/MB/GB/TB based on that max, so small test data shows as KB and real data as GB/TB. Returns `{name, divisor}`. |
| `_storageValues(data)` | Divides each month's bytes by the chosen unit's divisor. |
| `_storageSeries(data)` | The legend label reflecting the unit, e.g. `["Storage (GB)"]`. |

### Format helpers

| Helper | What it does |
|--------|--------------|
| `_typeLabels(data)` | Maps each mime-type bucket through `_friendlyType` for the pie legend. |
| `_friendlyType(mime)` | Turns `"application/pdf"` into `"PDF"`, `"image/png"` into `"PNG Image"`, etc.; for anything unknown, uppercases the part after the slash. |

### Migration helpers

| Helper | What it does |
|--------|--------------|
| `_migrationLabels()` | `["Migrated", "Non-Migrated"]`. |
| `_nonMigrated(total, migrated)` | `total − migrated`, floored at 0. |
| `_migrationValues(total, migrated)` | `[[migrated, nonMigrated]]` for the donut. |
| `_migratedPct(total, migrated)` | The whole-number percentage shown in the donut center. |

### KPI formatting

| Helper | What it does |
|--------|--------------|
| `_formatCount(n)` | Abbreviates big numbers: 1,200 → "1.2K", millions → "M", billions → "B". |
| `_formatStorage(bytes)` | Abbreviates bytes up through KB/MB/GB/TB/PB. |

---

## 11. The script as a whole — the big picture

The element works as a **reactive pipeline**:

1. **A date drives everything.** The user picks a period (or custom dates) →
   `_computeDates` sets `startDate`/`endDate`.

2. **Changing those dates auto-triggers every data element** — the five
   aggregations refill their chart arrays, and the two page-providers refill
   `totalCount`/`migratedCount`. You never manually call a fetch; the bindings do
   it.

3. **The helpers transform raw buckets into chart-ready shapes** — labels,
   values, units, percentages — purely as functions of the data, so when data
   changes the charts redraw automatically.

4. **Two query mechanisms, each chosen for scale:**
   - **Aggregations** for the charts (ES summarizes server-side, returns a
     handful of buckets).
   - **Page-provider `resultsCount`** for the migration counts (counts matches
     without downloading documents).
   - Both work the same at 300 docs or a billion.

5. **The view is just a function of state** — `activeView` decides which chart
   shows; `selectedPeriod` decides the dates; the data properties decide the
   numbers. Change any one and the relevant part of the UI re-renders on its own.

That's the whole system:

> **dates in → data elements fetch → helpers reshape → charts and KPIs render**

with CSS giving the white-card-on-grey, 3-per-row, responsive layout around it.

---

## Appendix — Why two query mechanisms (scale design)

This dashboard is built for repositories with up to a **billion documents**. Two
different data mechanisms are used, each chosen for how it scales:

**1. Charts use `nuxeo-repository-data` aggregations** (`grouped-by`,
`with-date-intervals`, `sum(...)`). These are Elasticsearch aggregations — ES
computes them across the whole index and returns only the buckets (a handful of
rows), never the documents. This scales to a billion docs by design.

**2. Migration counts use `nuxeo-page-provider` with NXQL and `page-size="1"`,**
reading only `resultsCount`. This returns a single filtered count without
fetching documents, so it also scales.

We deliberately do **not** count migrated docs with a term aggregation
(`grouped-by` on `systemName`): at scale that would need one bucket per distinct
source system and, capped by `group-limit`, would silently undercount. A filtered
`resultsCount` is exact and scale-safe.

We also avoid the `where` attribute on `nuxeo-repository-data` entirely: on this
build it crashes on ranges / `LIKE` / `IS NOT NULL`. NXQL inside a page provider
supports the complex-field `IS NOT NULL` predicate correctly and fails gracefully
instead of taking the page down.
