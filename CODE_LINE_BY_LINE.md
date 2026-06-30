# fid-analytical-dashboard — Complete Line-by-Line Walkthrough

This document explains **every line** of `fid-analytical-dashboard.html`. When a
line uses a CSS class, the matching rule from `fid-dashboard-styles.html` is
brought here and explained on the spot.

At the end, **one worked example** is traced through the whole file, showing the
intermediate result after each step.

**The running example** (used throughout): today is **2026-06-29**, the period is
the default **6months**, the repository has **296 documents** created in that
window (all in June), totalling **471,654 bytes** of file content, and **none are
migrated**.

---

# PART 1 — THE TOP COMMENT AND IMPORTS

```html
<!--
`fid-analytical-dashboard`
@group Nuxeo UI
@element fid-analytical-dashboard
... (description + scalability notes) ...
-->
```

Lines 1–46 are a comment block. It documents what the element is, its layout, and
the two scalability mechanisms (aggregations for charts, page-providers for
counts). Comments do nothing at runtime; they exist so the next developer
understands the design without reading all the code.

```html
<link rel="import" href="fid-dashboard-styles.html">
```

**Line 49.** An HTML Import — Polymer 2's way of pulling in another file. This
loads `fid-dashboard-styles.html`, which defines the CSS module. Without this
line, the styles wouldn't be available and the dashboard would render unstyled.

```html
<dom-module id="fid-analytical-dashboard">
  <template>
```

**Lines 51–52.** `dom-module` declares a custom element definition; its `id`
**must** match the `is:` name in the script (line 419) and the Nuxeo route. The
`<template>` holds the element's HTML, which Polymer stamps into the page for each
instance.

```html
<style include="iron-flex iron-flex-alignment nuxeo-styles fid-dashboard-styles"></style>
```

**Line 54.** Pulls in four style modules by id:
- `iron-flex`, `iron-flex-alignment` — Nuxeo's flexbox helper classes (e.g. the
  `flex` class used on line 186).
- `nuxeo-styles` — global theme variables like `--nuxeo-text-default`.
- `fid-dashboard-styles` — our own CSS (the imported file).

This is the bridge between the two files: the import (line 49) loads the file, and
this `include` activates its styles inside this element.

---

# PART 2 — THE DATA ELEMENTS (invisible; they fetch numbers)

These elements have no visible output. They run queries and write the results into
properties. They sit at the top so they initialise with the element.

## 2a. Documents-per-month aggregation

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

**Lines 62–69.**
- `start-date="[[startDate]]"` — **one-way** binding (square brackets): the
  element reads the `startDate` property.
- `end-date="[[_extendEndDate(endDate)]]"` — reads `endDate`, but **passes it
  through the function** `_extendEndDate` first (explained at line 521) so the end
  of *today* is included.
- `with-date-intervals="month"` — tells Elasticsearch to bucket results **by
  month** (a date histogram).
- `date-field="dc:created"` — bucket on the document creation date.
- `data="{{docsPerMonth}}"` — **two-way** binding (curly braces): the element
  writes its result array **back** into the `docsPerMonth` property.
- `index="[[index]]"` — which ES index to query (default `'nuxeo'`).

**Result shape:** `docsPerMonth` becomes something like
`[{key:"2026-06-01", value:296}, ...]` — one bucket per month.

## 2b. Storage-per-month aggregation

```html
<nuxeo-repository-data
  start-date="[[startDate]]"
  end-date="[[_extendEndDate(endDate)]]"
  with-date-intervals="month"
  date-field="dc:created"
  metrics="sum(file:content.length)"
  data="{{storagePerMonth}}"
  index="[[index]]">
</nuxeo-repository-data>
```

**Lines 72–80.** Same as 2a, but adds `metrics="sum(file:content.length)"` — for
each month bucket, ES **sums the file sizes** (in bytes) instead of just counting.
Writes into `storagePerMonth`. Result: `[{key:"2026-06-01", value:471654}, ...]`.

## 2c. By-Doc-Type aggregation

```html
<nuxeo-repository-data
  start-date="[[startDate]]"
  end-date="[[_extendEndDate(endDate)]]"
  grouped-by="ecm:primaryType"
  group-limit="10"
  data="{{byDocType}}"
  index="[[index]]">
</nuxeo-repository-data>
```

**Lines 83–90.** Instead of a date histogram, this uses
`grouped-by="ecm:primaryType"` — group documents by their **primary type** (File,
Note, Picture, etc.). `group-limit="10"` caps it at the top 10 types. Writes into
`byDocType`. Result: `[{key:"File", value:200}, {key:"Note", value:96}, ...]`.

## 2d. By-Format aggregation

```html
<nuxeo-repository-data ... grouped-by="file:content.mime-type" group-limit="10"
  data="{{byFormat}}" index="[[index]]">
```

**Lines 93–100.** Same idea, grouped by **mime-type** (`application/pdf`,
`image/png`, …). Writes into `byFormat`.

## 2e. Total-storage aggregation

```html
<nuxeo-repository-data ... metrics="sum(file:content.length)"
  data="{{totalStorageBytes}}" index="[[index]]">
```

**Lines 103–109.** No `with-date-intervals`, so it does **not** bucket by month —
it returns a single total: the sum of all file sizes in the period. Writes into
`totalStorageBytes` (a number). Result: `471654`.

## 2f. Total-documents count (page-provider)

```html
<nuxeo-page-provider
  auto
  page-size="1"
  query="[[_totalQuery(startDate, endDate)]]"
  current-page="{{_ignoredTotalPage}}"
  results-count="{{totalCount}}">
</nuxeo-page-provider>
```

**Lines 125–131.**
- `auto` — run automatically whenever the bound inputs change.
- `page-size="1"` — fetch at most **1 document**. We don't want documents, only
  the count, so we keep the page tiny.
- `query="[[_totalQuery(startDate, endDate)]]"` — the **full NXQL string** built
  by `_totalQuery` (line 532). No `?` placeholders.
- `current-page="{{_ignoredTotalPage}}"` — the returned page of documents. The
  element requires this binding, but we ignore the value.
- `results-count="{{totalCount}}"` — **the number we want**: the total count of
  matching documents, written into `totalCount`.

**Critical detail:** the attribute is `results-count` (property `resultsCount`),
**not** `number-of-results`. Using the wrong name leaves the count at 0.

## 2g. Migrated-documents count (page-provider)

```html
<nuxeo-page-provider ... query="[[_migratedQuery(startDate, endDate)]]"
  current-page="{{_ignoredMigratedPage}}" results-count="{{migratedCount}}">
```

**Lines 134–140.** Same as 2f, but uses `_migratedQuery` (which adds the
migration filter) and writes into `migratedCount`.

## 2h. Per-month counts (the dom-repeat)

```html
<template is="dom-repeat" items="[[_months]]" as="m">
  <!-- Migrated docs in this month -->
  <nuxeo-page-provider
    auto
    page-size="1"
    query="[[_monthMigratedQuery(m)]]"
    current-page="{{_ignoredMonthPage}}"
    results-count="{{_noopA}}"
    on-results-count-changed="_onMigratedCount">
  </nuxeo-page-provider>
  <!-- Total docs in this month -->
  <nuxeo-page-provider
    auto
    page-size="1"
    query="[[_monthTotalQuery(m)]]"
    current-page="{{_ignoredMonthPage2}}"
    results-count="{{_noopB}}"
    on-results-count-changed="_onTotalCount">
  </nuxeo-page-provider>
</template>
```

**Lines 154–173.** This is the per-month engine.
- `<template is="dom-repeat" items="[[_months]]" as="m">` — repeats its contents
  **once per item** in the `_months` array, exposing each item as `m`. So if
  there are 7 months, this stamps 7 copies of the two providers inside.
- For each month `m`, two providers run:
  - **Migrated:** `query="[[_monthMigratedQuery(m)]]"` builds the NXQL for that
    month's migrated docs (line 618). Its count fires
    `on-results-count-changed="_onMigratedCount"`.
  - **Total:** `query="[[_monthTotalQuery(m)]]"` builds the NXQL for that month's
    total docs (line 630). Its count fires `_onTotalCount`.
- `results-count="{{_noopA}}"` / `{{_noopB}}` — these are **throwaway** targets.
  We don't read the count from here; we read it from the **event**
  (`e.detail.value`) inside the handler. Why: a `{{m.migrated}}` binding inside
  `dom-repeat` writes silently without notifying Polymer, so the chart wouldn't
  update. Reading from the event and writing via `e.model.set` (in the handlers)
  is the notification-aware fix.

---

# PART 3 — THE PAGE STRUCTURE AND CSS

```html
<div class="page">
  <div class="content">
    <nuxeo-page>
```

**Lines 180–182.** Three nested wrappers. `nuxeo-page` is Nuxeo's page shell.

> **CSS for `.page` and `.content`** (styles file, lines 17–20):
> ```css
> .page, .content { @apply --layout-vertical; }
> ```
> `@apply --layout-vertical` is a Polymer mixin meaning "lay children out in a
> vertical column" (like `display:flex; flex-direction:column`).

```html
<div slot="header">
  <span class="flex">Analytical Dashboard</span>
</div>
```

**Lines 185–187.** `slot="header"` places this into Nuxeo's dark admin bar at the
top. `class="flex"` comes from `iron-flex` and makes the span grow to fill the
bar.

```html
<div class="dashboard-container">
```

**Line 189.** The main grey backdrop.

> **CSS for `.dashboard-container`** (styles file, lines 23–28):
> ```css
> .dashboard-container {
>   padding: 16px 24px 32px 24px;
>   background: var(--nuxeo-page-background, #f0f2f5);
>   min-height: 100vh;
>   box-sizing: border-box;
> }
> ```
> Padding around the content; a theme-driven grey background (falling back to
> `#f0f2f5`); `min-height:100vh` fills the screen height; `box-sizing:border-box`
> makes the padding count *inside* the width so nothing overflows.

---

# PART 4 — THE HEADER CARD AND CSS

```html
<nuxeo-card class="dates">
  <div class="header-slot">
```

**Lines 192–193.** A Nuxeo card containing the header row. (`.dates` has no rule
of its own; it's just a hook.)

> **CSS for `.header-slot`** (styles file, lines 31–39):
> ```css
> .header-slot {
>   display: flex; flex-direction: row;
>   align-items: center; justify-content: space-between;
>   flex-wrap: nowrap; gap: 24px; padding: 8px 4px;
> }
> ```
> A horizontal row. `justify-content: space-between` pushes the **title to the
> far left and the controls to the far right**. `align-items:center` centres them
> vertically; `gap:24px` spaces the children.

```html
<div class="header-left">
  <h1>Analytical Dashboard</h1>
  <p>Monthly usage tracking, growth metrics and adoption overview</p>
</div>
```

**Lines 195–198.** The title block.

> **CSS** (styles file, lines 41–68):
> ```css
> .header-left { display:flex; flex-direction:column; gap:4px; flex:1; min-width:0; }
> .header-left h1 { font-size:20px; font-weight:500; color:var(--nuxeo-text-default,#212121);
>                   white-space:nowrap; overflow:hidden; text-overflow:ellipsis; ... }
> .header-left p  { font-size:12px; color:var(--nuxeo-text-light,#9e9e9e); ... }
> ```
> `flex:1` lets the title block expand and push the controls right. `min-width:0`
> is a flexbox fix that allows the text to truncate. The `white-space:nowrap` +
> `overflow:hidden` + `text-overflow:ellipsis` trio turns a too-long title into
> "Analytical Dash…" instead of wrapping.

```html
<div class="header-divider"></div>
```

**Line 200.** A thin vertical line between the title and controls.

> **CSS** (lines 70–75):
> ```css
> .header-divider { width:1px; height:40px; background:var(--nuxeo-border,#e0e0e0); flex-shrink:0; }
> ```
> A 1px-wide, 40px-tall grey line. `flex-shrink:0` stops it from being squished.

```html
<div class="header-controls">
```

**Line 202.**

> **CSS** (lines 77–88):
> ```css
> .header-controls { display:flex; flex-direction:row; align-items:flex-end; gap:16px;
>                    flex-shrink:0; flex-wrap:wrap; }
> .header-controls nuxeo-select { min-width:150px; }
> ```
> Holds the dropdown and date pickers in a row, bottom-aligned so labels and
> inputs line up at their base. The dropdown gets a minimum width so it doesn't
> collapse.

```html
<nuxeo-select
  label="Period"
  selected="{{selectedPeriod}}"
  options='["1month","2months","3months","4months","6months","8months","10months","1year","custom"]'
  on-selected-changed="_onPeriodChanged">
</nuxeo-select>
```

**Lines 203–208.** The period dropdown.
- `selected="{{selectedPeriod}}"` — two-way binds the chosen value to the
  `selectedPeriod` property.
- `options='[...]'` — the list of choices (a JSON array in the attribute).
- `on-selected-changed="_onPeriodChanged"` — also call `_onPeriodChanged` when
  the selection changes.

```html
<template is="dom-if" if="[[_isCustom(selectedPeriod)]]">
  <div class="custom-dates">
    <div class="date-field">
      <label>Start Date</label>
      <input type="date" value="{{customStart::change}}" on-change="_onCustomDateChanged" />
    </div>
    <div class="date-field">
      <label>End Date</label>
      <input type="date" value="{{customEnd::change}}" on-change="_onCustomDateChanged" />
    </div>
  </div>
</template>
```

**Lines 211–222.** The custom date pickers, shown **only** when the period is
"custom".
- `<template is="dom-if" if="[[_isCustom(selectedPeriod)]]">` — stamps its
  contents only when `_isCustom` returns true (line 480).
- `value="{{customStart::change}}"` — two-way binding that updates on the input's
  native `change` event (the `::change` part names the event).
- `on-change="_onCustomDateChanged"` — also call that handler on change.

> **CSS for `.custom-dates`, `.date-field`** (lines 91–129): stacks each
> label-over-input as a column, with a bordered date input that turns blue on
> focus (`:focus` rule with a soft `box-shadow` glow).

---

# PART 5 — THE KPI STRIP AND CSS

```html
<div class="flex-layout tight">
  <nuxeo-card class="full-width">
    <div class="kpi-strip">
```

**Lines 229–231.**

> **CSS for `.flex-layout` and `.tight`** (lines 181–190):
> ```css
> .flex-layout { display:flex; flex-wrap:wrap; margin:16px -8px 0 -8px; }
> .flex-layout.tight { margin-top:8px; }
> ```
> The card-grid container. The negative `-8px` side margins cancel the `8px`
> margins on the cards inside so they sit flush to the edges. `.tight` reduces the
> top gap (used here because the KPI strip sits right under the header).

> **CSS for the card sizing** (lines 192–200):
> ```css
> .flex-layout nuxeo-card { flex: 1 0 calc(33% - 16px); margin: 0 8px 16px 8px; box-sizing:border-box; }
> .flex-layout nuxeo-card.full-width { flex: 1 0 calc(100% - 16px); }
> ```
> **This is the rule that controls the whole grid.** A normal card takes about a
> third of the row; a card with `full-width` spans the whole row. The KPI card
> here uses `full-width`.

> **CSS for `.kpi-strip`** (lines 132–137):
> ```css
> .kpi-strip { display:flex; flex-direction:row; flex-wrap:wrap; align-items:stretch; }
> ```
> Lays the four KPI cells in a row of equal-height columns.

```html
<div class="kpi-cell">
  <span class="kpi-value">[[_formatCount(totalCount)]]</span>
  <span class="kpi-label">New documents in period</span>
</div>
```

**Lines 233–236.** The first KPI cell.
- `[[_formatCount(totalCount)]]` — one-way binding that runs `totalCount` through
  `_formatCount` (line 1000), turning e.g. `296` into `"296"` or `1500000` into
  `"1.5M"`.

> **CSS for `.kpi-cell`, `.kpi-value`, `.kpi-label`** (lines 139–165):
> ```css
> .kpi-cell { display:flex; flex-direction:column; justify-content:center; flex:1 1 0;
>             min-width:120px; padding:4px 20px; border-right:1px solid var(--nuxeo-border,#eee); }
> .kpi-cell:last-child { border-right:none; }
> .kpi-value { font-size:26px; font-weight:600; color:var(--nuxeo-text-default,#1a73e8); line-height:1.1; }
> .kpi-label { font-size:11px; color:var(--nuxeo-text-light,#9e9e9e); margin-top:4px; }
> ```
> Each cell is an equal-width column (`flex:1 1 0`) with a thin right divider;
> the last cell drops its divider (`:last-child`). `.kpi-value` is the big number,
> `.kpi-label` the small grey caption.

```html
<div class="kpi-cell">
  <span class="kpi-value">[[_formatStorage(totalStorageBytes)]]</span>
  <span class="kpi-label">Storage added in period</span>
</div>
```

**Lines 238–241.** Second cell: `totalStorageBytes` through `_formatStorage`
(line 1009) → e.g. `"460.7 KB"`.

```html
<div class="kpi-cell">
  <span class="kpi-value">[[migratedCount]]</span>
  <span class="kpi-label">From migrated use cases</span>
</div>
```

**Lines 243–246.** Third cell: shows `migratedCount` **directly** (no formatting)
— you wanted the exact number.

```html
<div class="kpi-cell">
  <span class="kpi-value">[[_nonMigrated(totalCount, migratedCount)]]</span>
  <span class="kpi-label">From non-migrated use cases</span>
</div>
```

**Lines 248–251.** Fourth cell: `_nonMigrated` (line 941) computes
`totalCount − migratedCount`.

---

# PART 6 — SLOT 1: THE HERO CHART (tabs) AND CSS

```html
<div class="flex-layout">
  <nuxeo-card class="full-width" heading="Growth Overview">
```

**Lines 258–261.** A second card grid; the hero card is `full-width` so it spans
the row. `heading=` is Nuxeo's card title.

```html
<div class="view-tabs">
  <span class$="[[_tabClass(activeView, 'overall')]]" on-tap="_selectOverall">Overall</span>
  <span class$="[[_tabClass(activeView, 'docType')]]" on-tap="_selectDocType">By Doc Type</span>
  <span class$="[[_tabClass(activeView, 'format')]]"  on-tap="_selectFormat">By Format</span>
  <span class$="[[_tabClass(activeView, 'size')]]"    on-tap="_selectSize">By Size</span>
</div>
```

**Lines 264–269.** The four tab pills.
- `class$="[[_tabClass(...)]]"` — note the **`$`**. That's Polymer's syntax for
  binding a whole **attribute** value. `_tabClass` (line 690) returns either
  `"view-tab"` or `"view-tab active"` depending on which tab is selected.
- `on-tap="_selectOverall"` — tapping calls the handler that sets `activeView`.

> **CSS for `.view-tabs` and `.view-tab`** (lines 203–229):
> ```css
> .view-tabs { display:flex; flex-direction:row; gap:4px; margin:4px 0 12px 0; flex-wrap:wrap; }
> .view-tab { font-size:12px; font-weight:500; color:var(--nuxeo-text-light,#777);
>             padding:6px 14px; border-radius:16px; cursor:pointer; user-select:none;
>             transition: background .15s, color .15s; }
> .view-tab:hover { background: var(--nuxeo-page-background,#f0f2f5); }
> .view-tab.active { background: var(--nuxeo-primary-color,#1a73e8); color:white; }
> ```
> Rounded pills (`border-radius:16px`); the active one fills blue with white text;
> smooth colour transition on hover/select. `cursor:pointer` shows a hand;
> `user-select:none` stops text selection on click.

```html
<template is="dom-if" if="[[_isView(activeView, 'overall')]]">
  <div class="chart-sub-title">Documents and storage added per month</div>
  <div class="chart-box">
    <chart-bar
      labels="[[_monthLabels(storagePerMonth)]]"
      values="[[_overallValues(storagePerMonth, docsPerMonth, totalCount)]]"
      series="[[_overallSeries(storagePerMonth)]]"
      options="[[_overallOptions(storagePerMonth)]]">
    </chart-bar>
  </div>
</template>
```

**Lines 272–282.** The Overall chart, shown only when `activeView === 'overall'`
(via `_isView`, line 685).
- `labels="[[_monthLabels(storagePerMonth)]]"` — x-axis month labels (line 722).
- `values="[[_overallValues(storagePerMonth, docsPerMonth, totalCount)]]"` — the
  two data series, storage + documents (line 813).
- `series="[[_overallSeries(storagePerMonth)]]"` — the two legend labels
  (line 826).
- `options="[[_overallOptions(storagePerMonth)]]"` — the Chart.js config object
  (line 836).

> **CSS for `.chart-sub-title` and `.chart-box`** (lines 232–261):
> ```css
> .chart-sub-title { font-size:13px; font-weight:500; color:var(--nuxeo-text-default,#555);
>                    margin:0 0 4px 4px; }
> .chart-box { position:relative; width:100%; height:380px; margin-top:8px; }
> chart-line, chart-bar, chart-pie { display:block; width:100% !important; height:100% !important; }
> ```
> **`.chart-box` gives the chart a fixed 380px height** — Chart.js needs a
> fixed-height parent (with `maintainAspectRatio:false` in the options) or it
> collapses. The chart then fills that box 100%.

```html
<template is="dom-if" if="[[_isView(activeView, 'docType')]]">
  <div class="chart-sub-title">Documents by type</div>
  <template is="dom-if" if="[[!_isEmpty(byDocType)]]">
    <div class="chart-box">
      <chart-pie labels="[[_labels(byDocType)]]" values="[[_values(byDocType)]]" options='...'>
      </chart-pie>
    </div>
  </template>
  <template is="dom-if" if="[[_isEmpty(byDocType)]]">
    <div class="chart-empty">No documents found in this period</div>
  </template>
</template>
```

**Lines 285–299.** The By-Doc-Type pie.
- Outer `dom-if`: only when the docType tab is active.
- Inner `dom-if` pair: if `byDocType` has data (`!_isEmpty`), show the pie;
  otherwise show the empty-state message.
- `labels="[[_labels(byDocType)]]"` — raw bucket keys (File, Note, …) via
  `_labels` (line 710).
- `values="[[_values(byDocType)]]"` — bucket values via `_values` (line 716).
- The `options='{...}'` is inline JSON (a pie config: right-positioned legend, no
  animation).

> **CSS for `.chart-empty`** (lines 347–354):
> ```css
> .chart-empty { display:flex; align-items:center; justify-content:center;
>                height:240px; color:var(--nuxeo-text-light,#bbb); font-size:13px; }
> ```
> Centres the "No documents found" message in a 240px box.

```html
<template is="dom-if" if="[[_isView(activeView, 'format')]]">
  <div class="chart-sub-title">Documents by format</div>
  <template is="dom-if" if="[[!_isEmpty(byFormat)]]">
    <div class="chart-box">
      <chart-pie labels="[[_typeLabels(byFormat)]]" values="[[_values(byFormat)]]" options='...'>
      </chart-pie>
    </div>
  </template>
  <template is="dom-if" if="[[_isEmpty(byFormat)]]">
    <div class="chart-empty">No files found in this period</div>
  </template>
</template>
```

**Lines 302–316.** The By-Format pie. Same as By-Doc-Type, but labels come from
`_typeLabels` (line 902), which converts mime-types to friendly names.

```html
<template is="dom-if" if="[[_isView(activeView, 'size')]]">
  <div class="chart-sub-title">Storage added per month</div>
  <div class="chart-box">
    <chart-bar labels="[[_monthLabels(storagePerMonth)]]"
      values="[[_storageValues(storagePerMonth)]]"
      series="[[_storageSeries(storagePerMonth)]]" options='...'>
    </chart-bar>
  </div>
</template>
```

**Lines 319–329.** The By-Size storage bar (storage per month, single series).
- `values="[[_storageValues(storagePerMonth)]]"` — storage converted to the auto
  unit (line 883).
- `series="[[_storageSeries(storagePerMonth)]]"` — single legend label like
  `"Storage (KB)"` (line 893).

---

# PART 7 — SLOT 2: THE MIGRATION DONUT AND CSS

```html
<nuxeo-card heading="Migration Adoption">
  <div class="donut-wrap">
    <chart-pie
      labels="[[_migrationLabels()]]"
      values="[[_migrationValues(totalCount, migratedCount)]]"
      colors='["#2e7d32","#d32f2f"]'
      options='{... "cutoutPercentage":68 ...}'>
    </chart-pie>
    <div class="donut-center">
      <span class="pct">[[_migratedPct(totalCount, migratedCount)]]%</span>
      <span class="pct-label">MIGRATED</span>
    </div>
  </div>
  ...
```

**Lines 334–345.** This card has **no** `full-width`, so it's a one-third card.
- `labels="[[_migrationLabels()]]"` → `["Migrated","Non-Migrated"]` (line 936).
- `values="[[_migrationValues(totalCount, migratedCount)]]"` → `[[migrated,
  nonMigrated]]` (line 947).
- `colors='["#2e7d32","#d32f2f"]'` — green for migrated, red for non-migrated.
- `"cutoutPercentage":68` — cuts out 68% of the centre, turning the pie into a
  **donut**.
- `[[_migratedPct(...)]]%` — the big percentage in the hole (line 952).

> **CSS for `.donut-wrap` and `.donut-center`** (lines 264–296):
> ```css
> .donut-wrap { position:relative; width:100%; height:240px; margin-top:8px; }
> .donut-center { position:absolute; top:0; left:0; right:0; bottom:0;
>                 display:flex; flex-direction:column; align-items:center;
>                 justify-content:center; pointer-events:none; }
> .donut-center .pct { font-size:30px; font-weight:600; color:var(--nuxeo-text-default,#2e7d32); }
> .donut-center .pct-label { font-size:11px; color:var(--nuxeo-text-light,#9e9e9e); ... }
> ```
> `.donut-wrap` is `position:relative` so `.donut-center` can be `absolute`
> **inside it**, stretched to all four edges and flex-centred — that floats the
> "0% MIGRATED" text in the donut hole. `pointer-events:none` lets mouse hovers
> pass through to the chart so tooltips still work.

```html
<div class="migration-summary">
  <div class="stat"><span class="num">[[migratedCount]]</span><span class="lbl">Migrated</span></div>
  <div class="stat"><span class="num">[[_nonMigrated(totalCount, migratedCount)]]</span><span class="lbl">Non-Migrated</span></div>
  <div class="stat"><span class="num">[[totalCount]]</span><span class="lbl">Total</span></div>
</div>
```

**Lines 347–360.** Three little stat columns under the donut: Migrated,
Non-Migrated, Total.

> **CSS for `.migration-summary` and `.stat`** (lines 298–322):
> ```css
> .migration-summary { display:flex; flex-direction:row; justify-content:center; gap:24px; margin-top:12px; }
> .migration-summary .stat { display:flex; flex-direction:column; align-items:center; }
> .migration-summary .stat .num { font-size:16px; font-weight:600; ... }
> .migration-summary .stat .lbl { font-size:11px; color:var(--nuxeo-text-light,#9e9e9e); ... }
> ```
> Three centred number/label columns spaced 24px apart.

---

# PART 8 — SLOT 3: THE MONTHLY MIGRATION CHART AND CSS

```html
<nuxeo-card class="full-width" heading="Monthly Migration">
  <div class="chart-sub-title">Migrated vs non-migrated documents per month</div>
  <template is="dom-if" if="[[!_monthlyEmpty(monthlyMigrated)]]">
    <div class="chart-box">
      <chart-bar
        labels="[[_monthlyLabels(monthlyMigrated)]]"
        values="[[_monthlyValues(monthlyMigrated)]]"
        series="[[_monthlySeries()]]"
        colors="[[_monthlyColors()]]"
        options='{...}'>
      </chart-bar>
    </div>
  </template>
  <template is="dom-if" if="[[_monthlyEmpty(monthlyMigrated)]]">
    <div class="chart-empty">No documents found in this period</div>
  </template>
</nuxeo-card>
```

**Lines 365–381.** The new chart, `full-width` so it gets its own row.
- Outer `dom-if` pair: show the chart if `monthlyMigrated` has rows
  (`!_monthlyEmpty`, line 967), else the empty message.
- `labels="[[_monthlyLabels(monthlyMigrated)]]"` — month labels (line 972).
- `values="[[_monthlyValues(monthlyMigrated)]]"` — two series: migrated +
  non-migrated per month (line 978).
- `series="[[_monthlySeries()]]"` → `["Migrated","Non-Migrated"]` (line 986).
- `colors="[[_monthlyColors()]]"` → green + red (line 991), matching the donut.

Reuses `.chart-sub-title`, `.chart-box`, `.chart-empty` (already explained).

---

# PART 9 — SLOTS 4–6: PLACEHOLDERS AND CSS

```html
<nuxeo-card heading="Slot 4">
  <div class="slot-placeholder">
    <iron-icon icon="icons:insert-chart"></iron-icon>
    <span class="placeholder-label">Coming soon</span>
  </div>
</nuxeo-card>
```

**Lines 384–405** (three near-identical cards: Slot 4, 5, 6). No `full-width`, so
each is a one-third card. Each shows a faded chart icon over "Coming soon".

> **CSS for `.slot-placeholder`** (lines 325–344):
> ```css
> .slot-placeholder { display:flex; flex-direction:column; align-items:center;
>                     justify-content:center; min-height:200px; gap:10px; }
> .slot-placeholder iron-icon { --iron-icon-width:40px; --iron-icon-height:40px; color:var(--nuxeo-border,#ddd); }
> .slot-placeholder .placeholder-label { font-size:12px; color:var(--nuxeo-text-light,#bbb); ... }
> ```
> Centres a 40px faded icon over the "Coming soon" caption; `min-height:200px`
> keeps these cards a similar height to the real ones.

```html
            </div>      <!-- closes .flex-layout (the card grid) -->
          </div>        <!-- closes .dashboard-container -->
        </nuxeo-page>
      </div>            <!-- closes .content -->
    </div>              <!-- closes .page -->
  </template>
```

**Lines 407–415.** Closing tags for all the wrappers opened earlier.

---

# PART 10 — RESPONSIVE CSS (bottom of the styles file)

```css
@media (max-width: 1100px) {
  .flex-layout nuxeo-card { flex: 1 0 calc(50% - 16px); }
}
@media (max-width: 768px) {
  .dashboard-container { padding: 12px 16px 24px 16px; }
  .flex-layout nuxeo-card { flex: 1 0 calc(100% - 16px); }
  .header-slot { flex-direction: column; align-items: flex-start; gap: 16px; }
  .header-divider { display: none; }
  .header-controls { width: 100%; flex-direction: column; align-items: flex-start; }
  .header-controls nuxeo-select { width: 100%; }
  .custom-dates { flex-direction: column; width: 100%; }
  .date-field input[type="date"] { width: 100%; }
  .kpi-cell { flex: 1 1 45%; border-right: none;
              border-bottom: 1px solid var(--nuxeo-border, #eee); padding: 12px 16px; }
  .chart-box { height: 300px; }
}
```

**Styles file, lines 357–402.** Two breakpoints:
- **Under 1100px** wide: cards become **half-width** (2 per row).
- **Under 768px** (phones): cards go **full-width** (1 per row); the header
  stacks vertically; the divider hides; the dropdown and date inputs go
  full-width; KPI cells switch from a right divider to a bottom divider and wrap
  two-per-row; charts shrink to 300px tall.

This is the entire responsive behaviour, driven mostly by changing the card
`flex` basis.

---

# PART 11 — THE SCRIPT, LINE BY LINE

```javascript
Polymer({
  is: 'fid-analytical-dashboard',
  behaviors: [Nuxeo.LayoutBehavior],
```

**Lines 418–420.** Registers the element. `is:` is the tag name (matches the
`dom-module` id). `behaviors: [Nuxeo.LayoutBehavior]` mixes in Nuxeo's shared
layout helpers that Web UI elements rely on.

```javascript
observers: [
  '_buildMonths(startDate, endDate)'
],
```

**Lines 422–424.** A **complex observer**: whenever `startDate` **or** `endDate`
changes, Polymer calls `_buildMonths(startDate, endDate)` (line 580). This is what
rebuilds the month buckets when the period changes.

## Properties (lines 426–469)

```javascript
selectedPeriod: { type: String, value: '6months', observer: '_onPeriodChanged' },
```
A string, default `'6months'`. `observer:` means any change calls
`_onPeriodChanged` (line 485).

```javascript
startDate: { type: String, notify: true },
endDate:   { type: String, notify: true },
```
`notify:true` makes the property fire a `*-changed` event when it changes, so the
data elements bound to it re-run.

```javascript
customStart: { type: String },
customEnd:   { type: String },
activeView:  { type: String, value: 'overall' },
index:       { type: String, value: 'nuxeo' },
```
Custom date strings; the active tab (default `'overall'`); the ES index.

```javascript
docsPerMonth:      { type: Array,  value: function() { return []; } },
storagePerMonth:   { type: Array,  value: function() { return []; } },
byDocType:         { type: Array,  value: function() { return []; } },
byFormat:          { type: Array,  value: function() { return []; } },
totalStorageBytes: { type: Number, value: 0 },
```
The chart data holders. **Array/object defaults must be a function** (`value:
function(){return [];}`) so each element instance gets its own array, not a shared
one. The numbers default to 0.

```javascript
totalCount:    { type: Number, value: 0 },
migratedCount: { type: Number, value: 0 },
```
The whole-period counts, default 0 so the KPI/donut render safely before data
arrives.

```javascript
_months:         { type: Array, value: function() { return []; } },
monthlyMigrated: { type: Array, value: function() { return []; } },
```
`_months` = the month buckets; `monthlyMigrated` = the rows the monthly chart
reads.

```javascript
_ignoredTotalPage:    { type: Array },
_ignoredMigratedPage: { type: Array },
_ignoredMonthPage:    { type: Array },
_ignoredMonthPage2:   { type: Array },
_noopA:               { type: Number },
_noopB:               { type: Number }
```
Throwaway bind targets. The `_ignored*` ones receive each provider's `current-page`
(required but unused). `_noopA`/`_noopB` receive the per-month `results-count` —
also unused, because we read those counts from the **event** instead.

## ready (lines 472–474)

```javascript
ready: function() {
  this._computeDates(this.selectedPeriod);
},
```
Runs **once** when the element initialises. Calls `_computeDates('6months')` so
the dashboard opens with a populated date range — which then triggers every data
element to fetch.

## Period logic

```javascript
_isCustom: function(period) {
  return period === 'custom';
},
```
**Lines 480–482.** True only when period is `'custom'`. Drives the date-picker
`dom-if`.

```javascript
_onPeriodChanged: function(newVal) {
  if (!newVal || newVal === 'custom') return;
  this._computeDates(newVal);
},
```
**Lines 485–488.** Runs on dropdown change. Guards: if empty or `'custom'`, do
nothing (custom is handled by the manual pickers). Otherwise compute the dates.

```javascript
_computeDates: function(period) {
  var monthMap = { '1month':1, ..., '1year':12 };
  var months = monthMap[period];
  if (!months) return;
  var end   = new Date();
  var start = new Date();
  start.setMonth(start.getMonth() - months);
  this.startDate = this._toISODate(start);
  this.endDate   = this._toISODate(end);
},
```
**Lines 491–505.** The core date math.
- `monthMap` maps each period option to a number of months.
- `months = monthMap[period]` looks it up; `if(!months) return` bails on unknown
  periods (like `'custom'`).
- `end = new Date()` is today; `start = new Date()` then
  `start.setMonth(start.getMonth() - months)` moves back N months (handles year
  rollover automatically).
- Stores both as `"YYYY-MM-DD"` strings. Assigning them triggers all bindings on
  `startDate`/`endDate`.

```javascript
_onCustomDateChanged: function() {
  if (this.customStart && this.customEnd) {
    this.startDate = this.customStart;
    this.endDate   = this.customEnd;
  }
},
```
**Lines 508–513.** When both custom inputs are filled, copy them into
`startDate`/`endDate`.

```javascript
_toISODate: function(date) {
  return date.toISOString().split('T')[0];
},
```
**Lines 516–518.** Turns a JS `Date` into `"YYYY-MM-DD"`. `toISOString()` gives
`"2026-06-29T08:00:00.000Z"`; `.split('T')[0]` keeps `"2026-06-29"`.

```javascript
_extendEndDate: function(date) {
  if (!date) return date;
  var d = new Date(date);
  d.setDate(d.getDate() + 1);
  d.setMilliseconds(d.getMilliseconds() - 1);
  return d.toISOString().split('T')[0];
},
```
**Lines 521–527.** Pushes the end to the last instant of the day.
- Guard if no date.
- Parse into a Date (midnight at the start of that day).
- `+1 day` → midnight tomorrow.
- `−1 ms` → `23:59:59.999` of the original day, so the aggregations include
  today's documents.

## NXQL query builders

```javascript
_totalQuery: function(startDate, endDate) {
  var range = this._dateRangeClause(startDate, endDate);
  if (!range) return '';
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " + range;
},
```
**Lines 532–537.** Builds the total-documents query.
- `range = this._dateRangeClause(...)` builds the date part.
- `if(!range) return ''` — if dates aren't ready, return an empty query so the
  provider's `auto` doesn't fire.
- The query counts live (`ecm:isVersion = 0`), non-trashed (`ecm:isTrashed = 0`)
  docs in the date range.

```javascript
_migratedQuery: function(startDate, endDate) {
  var range = this._dateRangeClause(startDate, endDate);
  if (!range) return '';
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " + range +
         " AND fid_migration:migrationHistory/*1/systemName IS NOT NULL";
},
```
**Lines 540–546.** Same as `_totalQuery`, plus the migrated filter:
`fid_migration:migrationHistory/*1/systemName IS NOT NULL` — the document has at
least one migration-history entry. `/*1/` is NXQL's "any element of the list"
syntax.

```javascript
_dateRangeClause: function(startDate, endDate) {
  if (!startDate || !endDate) return '';
  var start = this._toTimestamp(startDate, false);
  var end   = this._toTimestamp(endDate, true);
  return "AND dc:created BETWEEN TIMESTAMP '" + start +
         "' AND TIMESTAMP '" + end + "'";
},
```
**Lines 551–557.** Builds the shared date filter.
- If either date is missing, return `''` (this is what makes the query builders
  return empty and keeps the providers from firing prematurely).
- Format both ends as timestamps (start at `00:00:00`, end at `23:59:59`).
- Return `AND dc:created BETWEEN TIMESTAMP '...' AND TIMESTAMP '...'`. The
  `TIMESTAMP '...'` literal is the correct NXQL form — bare dates or `?`
  placeholders both failed earlier.

```javascript
_toTimestamp: function(date, isEnd) {
  if (!date) return date;
  var day = String(date).split('T')[0];
  return day + (isEnd ? ' 23:59:59' : ' 00:00:00');
},
```
**Lines 561–565.** Turns `"2026-06-29"` into `"2026-06-29 23:59:59"` (end) or
`"2026-06-29 00:00:00"` (start).

## Per-month buckets

```javascript
_buildMonths: function(startDate, endDate) {
  if (!startDate || !endDate) { this._months = []; return; }
  var start = new Date(startDate);
  var end   = new Date(endDate);
  var months = [];
  var cursor = new Date(start.getFullYear(), start.getMonth(), 1);
  var last   = new Date(end.getFullYear(), end.getMonth(), 1);
  while (cursor <= last) {
    var y = cursor.getFullYear();
    var mo = cursor.getMonth();
    var firstOfMonth = new Date(y, mo, 1);
    var lastOfMonth  = new Date(y, mo + 1, 0);
    var mStart = firstOfMonth < start ? start : firstOfMonth;
    var mEnd   = lastOfMonth  > end   ? end   : lastOfMonth;
    months.push({
      key:      this._toISODate(firstOfMonth),
      label:    this._toMonthYear(this._toISODate(firstOfMonth)),
      start:    this._toTimestamp(this._toISODate(mStart), false),
      end:      this._toTimestamp(this._toISODate(mEnd), true),
      migrated: 0,
      all:      0
    });
    cursor.setMonth(cursor.getMonth() + 1);
  }
  this._months = months;
  this._refreshMonthly();
},
```
**Lines 580–615.** Splits the period into month buckets.
- Guard: no dates → empty `_months`, stop.
- `cursor` starts at the **first of the start month**; `last` is the **first of
  the end month**.
- The `while (cursor <= last)` loop runs once per month:
  - `firstOfMonth` = the 1st; `lastOfMonth = new Date(y, mo+1, 0)` = **day 0 of
    next month** = the last day of this month (a JS Date trick).
  - `mStart`/`mEnd` **clamp** to the actual selected range at the first and last
    months (so a 6-month window starting mid-month doesn't over-count).
  - Push a descriptor: `key` (first-of-month ISO), `label` ("Jun-26"), `start`
    and `end` timestamps for the query, and `migrated:0, all:0` placeholders.
  - `cursor.setMonth(cursor.getMonth()+1)` advances to next month.
- Store `_months` and call `_refreshMonthly` to build initial (zero) rows.

```javascript
_monthMigratedQuery: function(m) {
  if (!m || !m.start || !m.end) return '';
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " +
         "AND dc:created BETWEEN TIMESTAMP '" + m.start +
         "' AND TIMESTAMP '" + m.end + "' " +
         "AND fid_migration:migrationHistory/*1/systemName IS NOT NULL";
},
```
**Lines 618–625.** Builds one month's **migrated** query, using that month's
`m.start`/`m.end` and the migration filter.

```javascript
_monthTotalQuery: function(m) {
  if (!m || !m.start || !m.end) return '';
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " +
         "AND dc:created BETWEEN TIMESTAMP '" + m.start +
         "' AND TIMESTAMP '" + m.end + "'";
},
```
**Lines 630–636.** Builds one month's **total** query (no migration filter). This
is why non-migrated comes purely from queries: total − migrated.

```javascript
_onMigratedCount: function(e) {
  if (!e || !e.model || !e.detail) return;
  var val = e.detail.value;
  if (typeof val !== 'number') return;
  e.model.set('m.migrated', val);
  this._refreshMonthly();
},
```
**Lines 645–651.** Fires when a per-month **migrated** provider reports its count.
- Guards on the event shape.
- `val = e.detail.value` — the count from the event.
- `e.model.set('m.migrated', val)` — **the key line**. `e.model` is the
  `dom-repeat` row's model; `.set('m.migrated', val)` writes the value onto the
  correct month **and notifies Polymer** (a plain `m.migrated = val` would not
  notify, leaving the chart stale).
- `this._refreshMonthly()` rebuilds the chart rows.

```javascript
_onTotalCount: function(e) {
  if (!e || !e.model || !e.detail) return;
  var val = e.detail.value;
  if (typeof val !== 'number') return;
  e.model.set('m.all', val);
  this._refreshMonthly();
},
```
**Lines 653–659.** Same, for the per-month **total** provider, writing `m.all`.

```javascript
_refreshMonthly: function() {
  var rows = (this._months || []).map(function(m) {
    var migrated = m.migrated || 0;
    var all = m.all || 0;
    if (migrated > all) all = migrated;
    return {
      key:         m.key,
      label:       m.label,
      migrated:    migrated,
      nonMigrated: all - migrated,
      all:         all
    };
  });
  this.monthlyMigrated = rows;
},
```
**Lines 664–678.** Builds the chart rows from `_months`.
- For each month: read `migrated` and `all` (guarded to 0).
- `if (migrated > all) all = migrated` — safety so non-migrated never goes
  negative.
- Build a row with `nonMigrated = all − migrated`.
- Assign the new array to `monthlyMigrated` (a fresh array reference, which
  notifies the chart binding).

## Tab logic

```javascript
_isView: function(active, view) { return active === view; },
```
**Lines 685–687.** True when a given view is active; controls which chart `dom-if`
renders.

```javascript
_tabClass: function(active, view) {
  return active === view ? 'view-tab active' : 'view-tab';
},
```
**Lines 690–692.** Returns the pill's class string (`active` added when selected).

```javascript
_selectOverall: function() { this.activeView = 'overall'; },
_selectDocType: function() { this.activeView = 'docType'; },
_selectFormat:  function() { this.activeView = 'format'; },
_selectSize:    function() { this.activeView = 'size'; },
```
**Lines 694–697.** Each tap handler sets `activeView`, which instantly swaps the
visible chart and the highlighted pill.

## Generic chart helpers

```javascript
_isEmpty: function(data) { return !data || !data.length; },
```
**Lines 705–707.** True if an aggregation came back empty.

```javascript
_labels: function(data) {
  if (!data || !data.length) return [];
  return data.map(function(entry) { return entry.key; });
},
```
**Lines 710–713.** Pulls the `key` from each bucket → the axis/pie labels.

```javascript
_values: function(data) {
  if (!data || !data.length) return [[]];
  return [data.map(function(entry) { return entry.value; })];
},
```
**Lines 716–719.** Pulls the `value` from each bucket, wrapped in an **outer**
array because chart-elements expects an array of series: `[[v1,v2,...]]`.

```javascript
_monthLabels: function(data) {
  if (!data || !data.length) return [];
  var self = this;
  return data.map(function(entry) { return self._toMonthYear(entry.key); });
},
```
**Lines 722–726.** Friendly month labels. `var self = this` captures `this` so the
inner function can call `self._toMonthYear`.

```javascript
_toMonthYear: function(key) {
  if (key === null || typeof key === 'undefined' || key === '') return '';
  var months = ['Jan',...,'Dec'];
  var str = String(key);
  if (/^\d{10,}$/.test(str)) {           // epoch milliseconds
    var dms = new Date(parseInt(str, 10));
    if (!isNaN(dms.getTime())) return months[dms.getMonth()] + '-' + String(dms.getFullYear()).slice(-2);
  }
  var datePart = str.split('T')[0];      // strip time from ISO datetime
  var parts = datePart.split('-');
  if (parts.length >= 2) {
    var year = parts[0];
    var monthIdx = parseInt(parts[1], 10) - 1;
    if (monthIdx >= 0 && monthIdx <= 11 && year.length === 4)
      return months[monthIdx] + '-' + year.slice(-2);
  }
  var d = new Date(str);                  // last-resort parse
  if (!isNaN(d.getTime())) return months[d.getMonth()] + '-' + String(d.getFullYear()).slice(-2);
  return str;
},
```
**Lines 733–763.** Normalises any month-key format to `"Mon-YY"`. Handles three
shapes: epoch milliseconds (`/^\d{10,}$/`), dash-dates / ISO datetimes (split on
`-`, taking the date part before `T`), and a final `new Date()` fallback. The
hardening exists because the histogram's key format can vary, and we wanted the
labels to be reliable.

## Overall-chart helpers

```javascript
_docsAlignedToStorage: function(storage, docs) {
  var docMap = {};
  (docs || []).forEach(function(entry) { docMap[entry.key] = entry.value; });
  return (storage || []).map(function(entry) { return docMap[entry.key] || 0; });
},
```
**Lines 775–781.** Lines up the document counts to the **same month keys** as
storage. Build a lookup `{monthKey: docCount}`, then walk the storage months and
pull each month's doc count (0 if missing). This guarantees both series share an
x-axis.

```javascript
_reconcileDocs: function(rawMonthly, authoritativeTotal) {
  var aggTotal = rawMonthly.reduce(function(s, v){ return s + v; }, 0);
  if (!authoritativeTotal || aggTotal === 0 || aggTotal === authoritativeTotal) return rawMonthly;
  var factor = authoritativeTotal / aggTotal;
  var scaled = rawMonthly.map(function(v){ return Math.round(v * factor); });
  var scaledTotal = scaled.reduce(function(s, v){ return s + v; }, 0);
  var drift = authoritativeTotal - scaledTotal;
  if (drift !== 0 && scaled.length) {
    var maxIdx = 0;
    for (var i = 1; i < scaled.length; i++) if (scaled[i] > scaled[maxIdx]) maxIdx = i;
    scaled[maxIdx] += drift;
    if (scaled[maxIdx] < 0) scaled[maxIdx] = 0;
  }
  return scaled;
},
```
**Lines 789–808.** Scales the per-month document counts so they sum to the
authoritative KPI total.
- `aggTotal` = sum of the raw monthly counts.
- If there's nothing to reconcile (no total, empty, or already equal), return as-is.
- `factor` = target ÷ actual; multiply each month by it and round.
- Rounding can leave the sum off by a unit or two (`drift`); add that drift to the
  **busiest month** so the displayed numbers add up exactly. (This is the
  "reconcile" that keeps the chart total equal to the KPI.)

```javascript
_overallValues: function(storage, docs, total) {
  if (!storage || !storage.length) return [[]];
  var unit = this._storageUnit(storage);
  var storageVals = storage.map(function(entry) {
    var v = (entry.value || 0) / unit.divisor;
    return Math.round(v * 100) / 100;
  });
  var rawDocs = this._docsAlignedToStorage(storage, docs);
  var docVals = this._reconcileDocs(rawDocs, total);
  return [storageVals, docVals];
},
```
**Lines 813–823.** Builds the Overall chart's two series.
- Choose a storage unit (KB/MB/GB/TB).
- `storageVals` = each month's bytes ÷ divisor, rounded to 2 decimals.
- `rawDocs` = aligned document counts; `docVals` = those reconciled to `total`.
- Return `[storageVals, docVals]` — storage first, documents second (both bars).

```javascript
_overallSeries: function(storage) {
  return ['Storage (' + this._storageUnit(storage).name + ')', 'Documents'];
},
```
**Lines 826–828.** The two legend labels, e.g. `["Storage (KB)", "Documents"]`.

```javascript
_overallOptions: function(storage) {
  return {
    responsive: true, maintainAspectRatio: false, animation: false,
    legend: { display:true, position:'top', labels:{...} },
    scales: {
      xAxes: [{ gridLines:{display:false,drawBorder:false}, ticks:{autoSkip:false,maxRotation:30,...} }],
      yAxes: [{ gridLines:{color:'rgba(0,0,0,0.04)',drawBorder:false}, ticks:{beginAtZero:true,...} }]
    },
    tooltips: { mode:'index', intersect:false }
  };
},
```
**Lines 836–858.** The Chart.js config object.
- `responsive` + `maintainAspectRatio:false` let the chart fill the fixed 380px
  box.
- `animation:false` for snappy redraws.
- **One** y-axis (`yAxes` has a single entry) — deliberately, because a second
  unused axis falls back to a phantom 0–1 scale (a bug we hit). `beginAtZero:true`
  anchors it at 0.
- `tooltips:{mode:'index'}` shows both series' values when hovering a month.

## Storage helpers

```javascript
_storageMax: function(data) {
  if (!data || !data.length) return 0;
  return data.reduce(function(max, entry) {
    var v = entry.value || 0;
    return v > max ? v : max;
  }, 0);
},
```
**Lines 865–871.** Finds the largest monthly byte value, used to pick a unit.

```javascript
_storageUnit: function(data) {
  var max = this._storageMax(data);
  if (max >= 1099511627776) return { name:'TB', divisor:1099511627776 };
  if (max >= 1073741824)    return { name:'GB', divisor:1073741824 };
  if (max >= 1048576)       return { name:'MB', divisor:1048576 };
  return { name:'KB', divisor:1024 };
},
```
**Lines 874–880.** Picks KB/MB/GB/TB based on the max (thresholds are powers of
1024). Returns `{name, divisor}`. This auto-scaling is why tiny test data shows as
KB while real data shows as GB/TB.

```javascript
_storageValues: function(data) {
  if (!data || !data.length) return [[]];
  var unit = this._storageUnit(data);
  return [data.map(function(entry) {
    var v = (entry.value || 0) / unit.divisor;
    return Math.round(v * 100) / 100;
  })];
},
```
**Lines 883–890.** Each month's storage in the chosen unit, as a single series
`[[...]]` (for the standalone By-Size chart).

```javascript
_storageSeries: function(data) {
  return ['Storage (' + this._storageUnit(data).name + ')'];
},
```
**Lines 893–895.** Single legend label reflecting the unit.

## Format helpers

```javascript
_typeLabels: function(data) {
  if (!data || !data.length) return [];
  var self = this;
  return data.map(function(entry) { return self._friendlyType(entry.key); });
},
```
**Lines 902–906.** Maps each mime-type bucket through `_friendlyType` for the pie
legend.

```javascript
_friendlyType: function(mime) {
  if (!mime) return 'Unknown';
  var map = { 'application/pdf':'PDF', 'image/png':'PNG Image', ... };
  if (map[mime]) return map[mime];
  var parts = mime.split('/');
  return parts.length > 1 ? parts[1].toUpperCase() : mime;
},
```
**Lines 909–927.** Turns `"application/pdf"` into `"PDF"`, etc. Unknown types fall
back to the uppercased subtype (the part after `/`).

## Migration helpers

```javascript
_migrationLabels: function() { return ['Migrated', 'Non-Migrated']; },
```
**Lines 936–938.** The two donut labels.

```javascript
_nonMigrated: function(total, migrated) {
  var diff = (total || 0) - (migrated || 0);
  return diff > 0 ? diff : 0;
},
```
**Lines 941–944.** `total − migrated`, floored at 0.

```javascript
_migrationValues: function(total, migrated) {
  return [[migrated || 0, this._nonMigrated(total, migrated)]];
},
```
**Lines 947–949.** Donut data as one series of two slices: `[[migrated,
nonMigrated]]`.

```javascript
_migratedPct: function(total, migrated) {
  var t = total || 0;
  if (t <= 0) return 0;
  return Math.round(((migrated || 0) / t) * 100);
},
```
**Lines 952–956.** Migrated ÷ total × 100, rounded — the big donut percentage.
Guards divide-by-zero.

## Monthly-chart helpers

```javascript
_monthlyEmpty: function(rows) { return !rows || !rows.length; },
```
**Lines 967–969.** True when there are no month rows yet (drives the empty-state).

```javascript
_monthlyLabels: function(rows) {
  if (!rows || !rows.length) return [];
  return rows.map(function(r) { return r.label; });
},
```
**Lines 972–975.** One label per month (`"Jun-26"`).

```javascript
_monthlyValues: function(rows) {
  if (!rows || !rows.length) return [[]];
  var migrated    = rows.map(function(r) { return r.migrated || 0; });
  var nonMigrated = rows.map(function(r) { return r.nonMigrated || 0; });
  return [migrated, nonMigrated];
},
```
**Lines 978–983.** Two series: migrated per month and non-migrated per month.

```javascript
_monthlySeries: function() { return ['Migrated', 'Non-Migrated']; },
_monthlyColors: function() { return ['#2e7d32', '#d32f2f']; },
```
**Lines 986–993.** Legend labels and the green/red colours (matching the donut).

## KPI formatting

```javascript
_formatCount: function(n) {
  n = n || 0;
  if (n >= 1000000000) return (Math.round(n / 100000000) / 10) + 'B';
  if (n >= 1000000)    return (Math.round(n / 100000) / 10) + 'M';
  if (n >= 1000)       return (Math.round(n / 100) / 10) + 'K';
  return String(n);
},
```
**Lines 1000–1006.** Abbreviates big numbers with one decimal: thousands→K,
millions→M, billions→B. Below 1000, the raw number.

```javascript
_formatStorage: function(bytes) {
  bytes = bytes || 0;
  if (bytes >= 1125899906842624) return (... ) + ' PB';
  if (bytes >= 1099511627776)    return (... ) + ' TB';
  if (bytes >= 1073741824)       return (... ) + ' GB';
  if (bytes >= 1048576)          return (... ) + ' MB';
  if (bytes >= 1024)             return (... ) + ' KB';
  return bytes + ' B';
},
```
**Lines 1009–1017.** Same idea for bytes, scaling PB/TB/GB/MB/KB/B with one
decimal.

---

# PART 12 — ONE WORKED EXAMPLE, TRACED THROUGH EVERY STEP

**Setup:** today = **2026-06-29**, period = **6months** (default), repository has
**296 documents** all created in June 2026, totalling **471,654 bytes**, **0
migrated**.

We follow the data from element load to the rendered screen, showing the
**intermediate value after each step**.

### Step 1 — Element loads → `ready()`

`ready()` (line 472) runs `this._computeDates('6months')`.

### Step 2 — `_computeDates('6months')` (line 491)

- `monthMap['6months']` → `months = 6`
- `end = new Date()` → **2026-06-29**
- `start = new Date()` then `start.setMonth(month - 6)` → **2025-12-29**
- `this.startDate = _toISODate(start)` → **`"2025-12-29"`**
- `this.endDate = _toISODate(end)` → **`"2026-06-29"`**

> Intermediate: `startDate = "2025-12-29"`, `endDate = "2026-06-29"`.

### Step 3 — Assigning the dates fires two reactions in parallel

**(3a) The `observers` entry** `_buildMonths(startDate, endDate)` (line 422) runs.

**(3b) Every data element** bound to `startDate`/`endDate` re-runs its query.

### Step 3a — `_buildMonths("2025-12-29", "2026-06-29")` (line 580)

- `cursor` = first of start month = **2025-12-01**
- `last` = first of end month = **2026-06-01**
- The loop produces **7 months**: Dec-25, Jan-26, Feb-26, Mar-26, Apr-26, May-26,
  Jun-26.
- For each, it stores `key`, `label`, `start`/`end` timestamps, and
  `migrated:0, all:0`. The **first** month clamps its start to 2025-12-29 (the
  selected start), and the **last** clamps its end to 2026-06-29.

> Intermediate (`_months`, abbreviated):
> ```
> [ {label:"Dec-25", start:"2025-12-29 00:00:00", end:"2025-12-31 23:59:59", migrated:0, all:0},
>   {label:"Jan-26", start:"2026-01-01 00:00:00", end:"2026-01-31 23:59:59", migrated:0, all:0},
>   ... 
>   {label:"Jun-26", start:"2026-06-01 00:00:00", end:"2026-06-29 23:59:59", migrated:0, all:0} ]
> ```
> Then `_refreshMonthly()` runs, producing 7 rows all with `migrated:0,
> nonMigrated:0, all:0` (no counts yet). The monthly chart renders empty for now.

### Step 3a-continued — the dom-repeat fires 14 queries

`_months` now has 7 entries, so the `dom-repeat` (line 154) stamps **7 × 2 = 14**
page-providers. Each builds its query, e.g. for Jun-26:

- `_monthMigratedQuery(Jun)` →
  `SELECT * FROM Document WHERE ecm:isVersion = 0 AND ecm:isTrashed = 0 AND
  dc:created BETWEEN TIMESTAMP '2026-06-01 00:00:00' AND TIMESTAMP '2026-06-29
  23:59:59' AND fid_migration:migrationHistory/*1/systemName IS NOT NULL`
  → returns **0**.
- `_monthTotalQuery(Jun)` → same without the migration filter → returns **296**.

When the Jun total provider returns, `on-results-count-changed` fires
`_onTotalCount(e)`:
- `val = e.detail.value` → **296**
- `e.model.set('m.all', 296)` → writes 296 into the Jun entry of `_months` **and
  notifies Polymer**
- `_refreshMonthly()` rebuilds the rows.

> Intermediate (`monthlyMigrated`, Jun row): `{label:"Jun-26", migrated:0,
> nonMigrated:296, all:296}`. All other months: 0/0/0.

### Step 3b — the aggregations return (in parallel)

- `docsPerMonth` → `[{key:"2026-06-01", value:296}]` (only June has docs)
- `storagePerMonth` → `[{key:"2026-06-01", value:471654}]`
- `byDocType` → e.g. `[{key:"File", value:296}]`
- `byFormat` → e.g. `[{key:"application/pdf", value:296}]`
- `totalStorageBytes` → `471654`

### Step 3b — the page-provider counts return

- `_totalQuery(...)` → `totalCount = 296`
- `_migratedQuery(...)` → `migratedCount = 0`

### Step 4 — the KPI strip computes

- Cell 1: `_formatCount(296)` → `296 < 1000` → **`"296"`**
- Cell 2: `_formatStorage(471654)` → `471654 ≥ 1024` and `< 1048576` →
  `Math.round(471654/1024*10)/10` = `Math.round(4606.4)/10` = `460.6` →
  wait, precisely: `471654/1024 = 460.60`, `×10 = 4606.0`, `round = 4606`,
  `/10 = 460.6` → **`"460.6 KB"`** (your screenshots showed ~460.7 with slightly
  different byte totals; the method is the same).
- Cell 3: `migratedCount` → **`0`**
- Cell 4: `_nonMigrated(296, 0)` → `296 − 0 = 296` → **`296`**

### Step 5 — the Overall chart computes (active tab)

- `_monthLabels(storagePerMonth)`: storage has one bucket `"2026-06-01"` →
  `_toMonthYear("2026-06-01")` → **`["Jun-26"]`**
- `_overallValues(storagePerMonth, docsPerMonth, 296)`:
  - `_storageUnit` max is 471654 → between 1024 and 1048576 → **KB**, divisor 1024
  - `storageVals` = `[Math.round(471654/1024*100)/100]` = `[460.6]`
  - `rawDocs = _docsAlignedToStorage(...)` → `[296]`
  - `docVals = _reconcileDocs([296], 296)` → aggTotal 296 equals total 296 →
    returned unchanged → `[296]`
  - returns **`[[460.6], [296]]`**
- `_overallSeries(...)` → **`["Storage (KB)", "Documents"]`**
- The chart draws two bars for Jun-26: a storage bar (~460.6) and a documents bar
  (296), on one shared y-axis.

### Step 6 — the donut computes

- `_migrationLabels()` → `["Migrated","Non-Migrated"]`
- `_migrationValues(296, 0)` → `[[0, 296]]` → all red (non-migrated)
- `_migratedPct(296, 0)` → `0/296 → 0` → centre shows **`0%`**
- Summary row: Migrated **0**, Non-Migrated **296**, Total **296**.

### Step 7 — the Monthly Migration chart computes

- `_monthlyEmpty(monthlyMigrated)` → rows exist → false → the chart shows.
- `_monthlyLabels(...)` → `["Dec-25","Jan-26","Feb-26","Mar-26","Apr-26","May-26","Jun-26"]`
- `_monthlyValues(...)`:
  - migrated per month → `[0,0,0,0,0,0,0]`
  - nonMigrated per month → `[0,0,0,0,0,0,296]`
  - returns `[[0,0,0,0,0,0,0], [0,0,0,0,0,0,296]]`
- The chart draws a single red bar at Jun-26 (height 296), all other months empty,
  green (migrated) absent. y-axis scales 0–300.

### Step 8 — the user clicks "By Doc Type"

`_selectDocType()` sets `activeView = 'docType'`. The Overall `dom-if` removes its
chart; the docType `dom-if` (now `_isView` true) stamps its pie, fed by
`_labels(byDocType)` → `["File"]` and `_values(byDocType)` → `[[296]]`.

### Step 9 — the user changes the period to "1month"

- `selectedPeriod` changes → `_onPeriodChanged('1month')` → not custom →
  `_computeDates('1month')` → new `startDate`/`endDate`.
- That reassignment **re-runs the entire pipeline**: `_buildMonths` rebuilds
  `_months` (now 1–2 months), the `dom-repeat` re-fires the per-month queries, all
  aggregations and counts refetch, and every chart/KPI updates — automatically.

---

# THE WHOLE THING IN ONE SENTENCE

A date drives everything: changing the period sets `startDate`/`endDate`, which
re-runs the aggregations (charts) and the page-provider counts (KPIs, donut,
per-month), whose results flow through small helper functions that shape them into
labels, values, units, and percentages — and the view re-renders on its own,
with the CSS giving the white-cards-on-grey, 3-per-row, responsive layout around
it.
