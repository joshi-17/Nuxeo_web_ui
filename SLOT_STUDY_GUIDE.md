# fid-analytical-dashboard — Slot-by-Slot Study Guide

This guide walks the dashboard **one slot at a time**. For each slot you get:

- the **CSS** that styles it (briefly, with focus on what matters),
- the **queries / data** feeding it,
- the **flow of helper calls** — which helper runs, what **input** it receives,
  what it returns, and how that return feeds the next helper,
- a **small worked example** with the intermediate result after each helper.

At the very end there is **one large example traced through the entire code**.

A note on two ideas you'll see everywhere:

- **Polymer data binding.** `[[x]]` is one-way (read only); `{{x}}` is two-way
  (the child can write back); `class$="[[...]]"` / `style$="[[...]]"` bind a whole
  HTML *attribute*. When a bound property changes, Polymer re-invokes any helper
  that takes it as an argument and repaints just that spot.
- **Two data mechanisms.** Charts use `nuxeo-repository-data` (Elasticsearch
  *aggregations* → return only small buckets). Counts use `nuxeo-page-provider`
  with `page-size="1"` reading `results-count` (a filtered count, no documents
  fetched). Both are chosen so the dashboard is safe at a billion documents.

---

# THE DATA LAYER (shared by all slots)

Before any slot renders, these invisible elements at the top of the template
fetch everything. They all bind to `startDate` / `endDate`, so when the period
changes they **all re-run automatically**.

### Aggregations (charts) — `nuxeo-repository-data`

| Element | What it groups / measures | Writes into |
|---|---|---|
| docs per month | count, `with-date-intervals="month"`, `date-field="dc:created"` | `docsPerMonth` |
| storage per month | `sum(file:content.length)` per month | `storagePerMonth` |
| by doc type | `grouped-by="ecm:primaryType"` (count) | `byDocType` |
| storage by doc type | `grouped-by="ecm:primaryType"` + `sum(file:content.length)` | `storageByDocType` |
| by format | `grouped-by="file:content.mime-type"` (count) | `byFormat` |
| total storage | `sum(file:content.length)` (no interval) | `totalStorageBytes` |

Each returns an array of buckets shaped like `[{key, value}, ...]`. Elasticsearch
computes across the whole index and returns only these buckets — never documents.

### Counts — `nuxeo-page-provider`

- **Total documents:** `query="[[_totalQuery(startDate, endDate)]]"`,
  `results-count="{{totalCount}}"`.
- **Migrated documents:** `query="[[_migratedQuery(startDate, endDate)]]"`,
  `results-count="{{migratedCount}}"`.
- **Per-month (dom-repeat over `_months`):** two providers per month — migrated
  (`_monthMigratedQuery(m)`) and total (`_monthTotalQuery(m)`) — feeding the
  count handlers `_onMigratedCount` / `_onTotalCount`.
- **Per-use-case (dom-repeat over `_useCaseTypes`):** one provider per type —
  migrated count (`_typeMigratedQuery(uc.type, startDate, endDate)`) — feeding
  `_onTypeMigratedCount`.

`page-size="1"` + reading `results-count` = "give me the match count, not the
documents." `current-page` is bound to an `_ignored*` property only because the
element requires it.

### The trigger chain (how a period change fans out)

1. `ready()` runs once → `_computeDates('1month')` sets `startDate`/`endDate`.
2. Assigning those fires **two things in parallel**:
   - every aggregation + count element bound to them **refetches**;
   - the `observers` entry `_buildMonths(startDate, endDate)` rebuilds `_months`.
3. Aggregation results land in `byDocType` / `storageByDocType`, which fire the
   other two observers `_computeTopByDocs` / `_computeTopByStorage`.
4. Each slot's helpers re-run against the fresh data and the view repaints.

---

# HEADER (title + period selector)

### Markup

```html
<nuxeo-card class="dates">
  <div class="header-slot">
    <div class="header-left"><h1>Analytical Dashboard</h1><p>Nuxeo usage tracking dashboard</p></div>
    <div class="header-divider"></div>
    <div class="header-controls">
      <div class="period-block">
        <div class="period-row">
          <nuxeo-select label="Period" selected="{{selectedPeriod}}"
            options='["1month","3months","6months","9months","1year","custom"]'
            on-selected-changed="_onPeriodChanged"></nuxeo-select>
          <div class="period-info-wrap">
            <iron-icon class="period-info-icon" icon="icons:info-outline"></iron-icon>
            <div class="period-tooltip">The period is calculated from today's date ...</div>
          </div>
        </div>
      </div>
      <template is="dom-if" if="[[_isCustom(selectedPeriod)]]">
        <div class="custom-dates">
          <div class="date-field"><label>Start Date</label><input type="date" value="{{customStart::change}}" /></div>
          <div class="date-field"><label>End Date</label><input type="date" value="{{customEnd::change}}" /></div>
          <div class="date-field"><label>&nbsp;</label>
            <button class="custom-submit" on-tap="_applyCustomDates">Apply</button></div>
        </div>
      </template>
    </div>
  </div>
</nuxeo-card>
```

### CSS (brief)

- `.header-slot` — a row with `justify-content: space-between`, so title sits
  left and controls right; `.header-divider` is the thin vertical line between.
- `.header-left h1/p` use `white-space:nowrap; overflow:hidden; text-overflow:
  ellipsis` so a long title truncates instead of wrapping.
- **Tooltip (the important bit):** `.period-tooltip` is `display:none` by default
  and `position:absolute; right:0` (anchored to the icon's right edge, opening
  **leftward** so it can't run off the screen). The rule
  `.period-info-wrap:hover .period-tooltip { display:block; }` is what makes it
  appear **only while hovering** the icon. `max-width:80vw` guarantees it fits.
- `.custom-submit` is the blue Apply button (hover/active shades).

### Flow of helpers

**On dropdown change** → `_onPeriodChanged(newVal)`:
- **input:** the newly selected string, e.g. `"6months"`.
- if it's empty or `"custom"`, it returns (custom is handled by the Apply
  button); otherwise it calls `_computeDates(newVal)`.

**`_computeDates(period)`:**
- **input:** e.g. `"6months"`.
- looks up `monthMap["6months"] → 6`; `end = today`; `start = today − 6 months`;
  writes `startDate`/`endDate` via `_toISODate`.
- **`_toISODate(date)` input:** a JS `Date`; **returns** `"YYYY-MM-DD"`.

**On custom Apply** → `_applyCustomDates()`:
- reads `customStart`/`customEnd` (bound from the two date inputs); if both are
  set, assigns them to `startDate`/`endDate`. Nothing recomputes until this runs.

**`_isCustom(period)`** — input the current period; returns `true` only for
`"custom"`, which is what makes the date-picker `dom-if` appear.

### Small example

User picks **6months**, today = 2026-06-29:
- `_onPeriodChanged("6months")` → not custom → `_computeDates("6months")`
- `monthMap["6months"] = 6` → `start = 2025-12-29`, `end = 2026-06-29`
- `startDate = "2025-12-29"`, `endDate = "2026-06-29"` → **everything refetches.**

---

# KPI STRIP (four headline numbers)

### Markup

```html
<div class="flex-layout tight">
  <nuxeo-card class="full-width">
    <div class="kpi-strip">
      <div class="kpi-cell"><span class="kpi-value">[[_formatCount(totalCount)]]</span>
        <span class="kpi-label">New documents created</span></div>
      <div class="kpi-cell"><span class="kpi-value">[[_formatStorage(totalStorageBytes)]]</span>
        <span class="kpi-label">of storage added</span></div>
      <div class="kpi-cell"><span class="kpi-value">[[migratedCount]]</span>
        <span class="kpi-label">From migrated use cases</span></div>
      <div class="kpi-cell"><span class="kpi-value">[[_nonMigrated(totalCount, migratedCount)]]</span>
        <span class="kpi-label">From non-migrated use cases</span></div>
    </div>
  </nuxeo-card>
</div>
```

### CSS (brief)

- `.flex-layout.tight` — the card grid container, with a smaller top margin so
  the strip tucks under the header.
- `.kpi-strip` is a flex row; each `.kpi-cell` is `flex:1 1 0` (equal widths)
  with a right divider; `:last-child` drops the divider.
- `.kpi-value` is the big blue number; `.kpi-label` the small grey caption.

### Data + flow

Two cells show raw properties (`migratedCount`, and `totalCount` indirectly);
two run through formatters.

- **`_formatCount(n)`** — input `totalCount` (a number). Returns a short string:
  `296 → "296"`, `1500 → "1.5K"`, `2_300_000 → "2.3M"`, `4e9 → "4B"`.
- **`_formatStorage(bytes)`** — input `totalStorageBytes`. Auto-scales:
  `471654 → "460.6 KB"`, `5_368_709_120 → "5 GB"`.
- **`_nonMigrated(total, migrated)`** — inputs `totalCount`, `migratedCount`.
  Returns `total − migrated`, floored at 0.

### Small example

`totalCount = 298`, `totalStorageBytes = 506388`, `migratedCount = 1`:
- Cell 1: `_formatCount(298)` → `"298"`
- Cell 2: `_formatStorage(506388)` → `506388 ≥ 1024` → `506388/1024 ≈ 494.5` → `"494.5 KB"`
- Cell 3: `migratedCount` → `1`
- Cell 4: `_nonMigrated(298, 1)` → `297`

---

# SLOT 1 — GROWTH OVERVIEW (hero, tabbed)

One `full-width` card with four tabs: **Overall, By Doc Type, By Format, By Size**.
`activeView` (default `"overall"`) decides which sub-chart shows.

### Markup (condensed)

```html
<nuxeo-card class="full-width" heading="Growth Overview">
  <div class="view-tabs">
    <span class$="[[_tabClass(activeView,'overall')]]" on-tap="_selectOverall">Overall</span>
    <span class$="[[_tabClass(activeView,'docType')]]" on-tap="_selectDocType">By Doc Type</span>
    <span class$="[[_tabClass(activeView,'format')]]"  on-tap="_selectFormat">By Format</span>
    <span class$="[[_tabClass(activeView,'size')]]"    on-tap="_selectSize">By Size</span>
  </div>

  <template is="dom-if" if="[[_isView(activeView,'overall')]]">
    <div class="chart-box">
      <chart-bar labels="[[_monthLabels(storagePerMonth)]]"
                 values="[[_overallValues(storagePerMonth, docsPerMonth, totalCount)]]"
                 series="[[_overallSeries(storagePerMonth)]]"
                 options="[[_overallOptions(storagePerMonth)]]"></chart-bar>
    </div>
  </template>
  <!-- docType / format / size tabs: pies + a storage bar, each guarded by dom-if -->
</nuxeo-card>
```

### CSS (brief)

- `.view-tab` — rounded pill; `.view-tab.active` fills blue with white text.
  `_tabClass` toggles the `active` class.
- `.chart-box` — **fixed 380px height**. Chart.js needs a fixed-height parent
  (with `maintainAspectRatio:false` in options) or it collapses to nothing.
- `chart-bar/pie` forced to fill `100%` of that box.

### Which data each tab uses

| Tab | Data | Label helper | Value helper |
|---|---|---|---|
| Overall | `storagePerMonth`, `docsPerMonth`, `totalCount` | `_monthLabels` | `_overallValues` |
| By Doc Type | `byDocType` | `_labels` | `_values` |
| By Format | `byFormat` | `_typeLabels` | `_values` |
| By Size | `storagePerMonth` | `_monthLabels` | `_storageValues` |

### Tab switching flow

- `_tabClass(active, view)` — inputs the current `activeView` and this tab's name;
  returns `"view-tab active"` or `"view-tab"`.
- `_selectOverall/_selectDocType/...` — set `activeView`; that instantly flips
  which `dom-if` renders (via `_isView(active, view)` → boolean).

### Overall tab — helper flow (the most involved chart)

**Labels:** `_monthLabels(storagePerMonth)` → maps each bucket key through
`_toMonthYear`. Input a bucket like `{key:"2026-06-01", value:...}`; output
`"Jun-26"`.

**Values:** `_overallValues(storage, docs, total)` builds the two bar series:
1. `_storageUnit(storage)` — input the storage buckets; finds the max via
   `_storageMax` and returns `{name, divisor}` (KB/MB/GB/TB).
2. storage series = each bucket's `value / divisor`, rounded to 2 dp.
3. `_docsAlignedToStorage(storage, docs)` — input both bucket arrays; returns a
   docs array lined up to the **storage month keys** (0 where a month has no
   docs), so both series share the x-axis.
4. `_reconcileDocs(rawDocs, total)` — input the aligned docs + `totalCount`;
   scales them so they **sum to the KPI total**, pushing rounding drift onto the
   busiest month. (Keeps the chart total equal to the KPI's "New documents.")
5. returns `[storageVals, docVals]`.

**Series (legend):** `_overallSeries(storage)` → `["Storage added in (KB)",
"Number of documents created"]` (unit reflects `_storageUnit`).

**Options:** `_overallOptions(storage)` → a Chart.js config with **one** shared
Y-axis (a second, unused axis would fall back to a phantom 0-1 scale), no
animation, index-mode tooltips.

### Small example (Overall)

`storagePerMonth = [{key:"2026-06-01", value:506388}]`,
`docsPerMonth = [{key:"2026-06-01", value:298}]`, `totalCount = 298`:
- `_monthLabels(...)` → `["Jun-26"]`
- `_storageUnit(...)` → max 506388 → between 1024 and 1048576 → `{name:"KB", divisor:1024}`
- storageVals → `[Math.round(506388/1024*100)/100] = [494.52]`
- `_docsAlignedToStorage(...)` → `[298]`
- `_reconcileDocs([298], 298)` → aggTotal 298 == total 298 → unchanged → `[298]`
- `_overallValues` → `[[494.52], [298]]`
- `_overallSeries` → `["Storage added in (KB)", "Number of documents created"]`
- Chart draws a ~494.52 storage bar and a 298 documents bar at Jun-26.

### By Doc Type / By Format — helper flow

- `_isEmpty(data)` — input the aggregation; if empty, the empty-state
  `dom-if` shows "No documents found" instead of the pie.
- **By Doc Type:** `_labels(byDocType)` → raw keys (`["File","Note",...]`);
  `_values(byDocType)` → `[[v1, v2, ...]]` (wrapped one level, as chart-elements
  expects an array of series).
- **By Format:** `_typeLabels(byFormat)` → each mime-type through `_friendlyType`
  (`"application/pdf" → "PDF"`); `_values(byFormat)` for the numbers.

Example (By Doc Type): `byDocType = [{key:"File",value:200},{key:"Note",value:98}]`
→ labels `["File","Note"]`, values `[[200,98]]` → two-slice pie.

### By Size — helper flow

- `_storageValues(storagePerMonth)` → one series `[[perMonthInUnit...]]`.
- `_storageSeries(storagePerMonth)` → `["Storage (KB)"]` (unit-aware).
- `_monthLabels` for the x-axis.


---

# SLOT 2 — MIGRATION ADOPTION (donut)

A `card-onethird` card: a green/red donut with a big % floating in the hole, and a
three-number summary below.

### Markup

```html
<nuxeo-card class="card-onethird" heading="Migration Adoption">
  <div class="donut-wrap">
    <chart-pie labels="[[_migrationLabels()]]"
               values="[[_migrationValues(totalCount, migratedCount)]]"
               colors='["#2e7d32","#d32f2f"]'
               options='{... "cutoutPercentage":68 ...}'></chart-pie>
    <div class="donut-center">
      <span class="pct">[[_migratedPct(totalCount, migratedCount)]]%</span>
      <span class="pct-label">MIGRATED</span>
    </div>
  </div>
  <div class="migration-summary">
    <div class="stat"><span class="num">[[migratedCount]]</span><span class="lbl">Migrated</span></div>
    <div class="stat"><span class="num">[[_nonMigrated(totalCount, migratedCount)]]</span><span class="lbl">Non-Migrated</span></div>
    <div class="stat"><span class="num">[[totalCount]]</span><span class="lbl">Total</span></div>
  </div>
</nuxeo-card>
```

### CSS (brief)

- **`.card-onethird`** → `flex: 0 0 calc(33.333% - 16px)` so it takes exactly a
  third of its row (Monthly Migration takes the other two-thirds). Because this
  row is its own `.flex-layout`, the third is exact.
- **`.donut-wrap`** is `position:relative` (240px tall). **`.donut-center`** is
  `position:absolute` stretched to all four edges and flex-centred — that's what
  floats the "0% MIGRATED" text in the donut hole. `pointer-events:none` lets
  mouse hovers pass through to the chart so tooltips still work.
- The donut shape itself comes from the chart option **`cutoutPercentage:68`**
  (cuts out 68% of the center).
- `.migration-summary .stat` — three centred number/label columns.

### Data + queries

This slot is driven by the two whole-period counts:
- `totalCount` from `_totalQuery(startDate, endDate)`.
- `migratedCount` from `_migratedQuery(startDate, endDate)`, whose NXQL adds
  `AND fid_migration:migrationHistory/*/systemName IS NOT NULL` — the `/*/`
  syntax is the one that works on this build (the `/*1/` form returned 0).

### Helper flow

- **`_migrationLabels()`** — no input; returns `["Migrated","Non-Migrated"]`
  (the two donut segment labels + legend).
- **`_migrationValues(total, migrated)`** — inputs `totalCount`, `migratedCount`;
  returns `[[migrated, nonMigrated]]` where nonMigrated comes from `_nonMigrated`.
  This one wrapped array is the donut's single data series (two slices).
- **`_nonMigrated(total, migrated)`** — returns `total − migrated`, floored at 0.
  Used both here and by the summary row and the KPI strip.
- **`_migratedPct(total, migrated)`** — inputs the two counts; returns
  `round(migrated / total × 100)`, guarding divide-by-zero. This is the big
  number in the hole.

### Small example

`totalCount = 298`, `migratedCount = 1`:
- `_migrationLabels()` → `["Migrated","Non-Migrated"]`
- `_nonMigrated(298, 1)` → `297`
- `_migrationValues(298, 1)` → `[[1, 297]]` → donut is almost entirely red with a
  thin green sliver
- `_migratedPct(298, 1)` → `round(1/298×100) = round(0.34) = 0` → center shows
  **"0%"** (correct — 1 of 298 rounds to 0%)
- summary row: Migrated **1**, Non-Migrated **297**, Total **298**.

---

# SLOT 3 — MONTHLY MIGRATION (grouped bars per month)

A `card-twothirds` card beside the donut: green (migrated) + red (non-migrated)
bars for each month in the period.

### Markup

```html
<nuxeo-card class="card-twothirds" heading="Monthly Migration">
  <div class="chart-sub-title">Migrated vs non-migrated documents per month</div>
  <template is="dom-if" if="[[!_monthlyEmpty(monthlyMigrated)]]">
    <div class="chart-box">
      <chart-bar labels="[[_monthlyLabels(monthlyMigrated)]]"
                 values="[[_monthlyValues(monthlyMigrated)]]"
                 series="[[_monthlySeries()]]"
                 colors="[[_monthlyColors()]]"
                 options='{...}'></chart-bar>
    </div>
  </template>
  <template is="dom-if" if="[[_monthlyEmpty(monthlyMigrated)]]">
    <div class="chart-empty">No documents found in this period</div>
  </template>
</nuxeo-card>
```

### CSS (brief)

- **`.card-twothirds`** → `flex: 0 0 calc(66.667% - 16px)`; sits beside the
  one-third donut on the same row.
- Reuses `.chart-box` (380px) and `.chart-empty` (centred placeholder).

### The data pipeline (this is the clever part)

The chart reads `monthlyMigrated`, an array of `{label, migrated, nonMigrated,
all}`. That array is assembled from **per-month queries**, not from the
aggregation. Here's the chain:

1. **`_buildMonths(startDate, endDate)`** (an observer) splits the period into
   month descriptors `{key, label, start, end, migrated:0, all:0}`. It parses
   dates with `_parseLocal` and formats with `_toISOLocal` (both local-time, to
   avoid a UTC off-by-one that used to drop the last month).
2. The template's `dom-repeat` over `_months` stamps **two page-providers per
   month**: one runs `_monthMigratedQuery(m)`, one runs `_monthTotalQuery(m)`.
3. When each returns, `_onMigratedCount(e)` / `_onTotalCount(e)` fire.
   - **input `e`:** the change event; `e.detail.value` is the count,
     `e.model` is the repeated row's model.
   - they write the value onto the right month with **`e.model.set('m.migrated',
     val)`** / `e.model.set('m.all', val)`. Using `e.model.set` (not a plain
     assignment) is essential: it **notifies Polymer** so observers react. A
     plain `{{m.all}}` mutation is silent and left the chart stale — that was a
     real bug.
   - then they call `_refreshMonthly()`.
4. **`_refreshMonthly()`** maps `_months` into chart rows: `nonMigrated = all −
   migrated` (guarded so it's never negative), assigns `monthlyMigrated`.

### Chart helpers

- `_monthlyEmpty(rows)` — input `monthlyMigrated`; true if empty → shows the
  empty-state instead of the chart.
- `_monthlyLabels(rows)` — returns each row's `label` (e.g. `["Dec-25",...,"Jun-26"]`).
- `_monthlyValues(rows)` — returns two series `[[migrated...], [nonMigrated...]]`.
- `_monthlySeries()` → `["Migrated","Non-Migrated"]`; `_monthlyColors()` →
  `["#2e7d32","#d32f2f"]` (green, red — matching the donut).

### Query helpers

- `_monthMigratedQuery(m)` — input a month descriptor; NXQL for docs created in
  `[m.start, m.end]` **with** the `/*/systemName IS NOT NULL` migrated filter.
- `_monthTotalQuery(m)` — same window **without** the migrated filter. So
  non-migrated = total − migrated, entirely from queries.

### Small example (6-month period, only June has data)

`_months` = 7 descriptors Dec-25 … Jun-26. For Jun-26, `m.start =
"2026-06-01 00:00:00"`, `m.end = "2026-06-29 23:59:59"`.
- Jun total provider returns 298 → `_onTotalCount` → `e.model.set('m.all', 298)`
  → `_refreshMonthly()`.
- Jun migrated provider returns 1 → `_onMigratedCount` → `e.model.set('m.migrated',
  1)` → `_refreshMonthly()`.
- `_refreshMonthly()` builds the Jun row `{label:"Jun-26", migrated:1,
  nonMigrated:297, all:298}`; all other months 0/0/0.
- `_monthlyLabels` → 7 labels; `_monthlyValues` → `[[0,0,0,0,0,0,1],
  [0,0,0,0,0,0,297]]` → tiny green + tall red bar at Jun-26.


---

# SLOTS 4 & 5 — TOP 5 USE CASES

Two `card-half` cards side by side. **Slot 4** ranks primary types by document
count; **Slot 5** ranks by storage added. Each row shows rank, name, a progress
bar, the value, and a **migrated %**.

### Markup (Slot 4; Slot 5 is identical but storage-based)

```html
<nuxeo-card class="card-half" heading="Top 5 Use Cases">
  <div class="usecase-subhead">by document volume</div>
  <template is="dom-if" if="[[!_isEmpty(topByDocs)]]">
    <div class="usecase-list">
      <div class="usecase-head-row">
        <span class="uc-rank-h">#</span><span class="uc-name-h">USE CASE</span>
        <span class="uc-val-h">DOCS</span><span class="uc-status-h">MIGRATED %</span>
      </div>
      <template is="dom-repeat" items="[[topByDocs]]" as="row">
        <div class="usecase-row">
          <span class="uc-rank">[[_inc(index)]]</span>
          <div class="uc-main">
            <span class="uc-name">[[row.label]]</span>
            <div class="uc-bar"><div class="uc-bar-fill" style$="[[_barStyle(row.value, topByDocsMax)]]"></div></div>
          </div>
          <span class="uc-val">[[_formatCount(row.value)]]</span>
          <span class="uc-status">[[_statusPct(row.type, _docCountByType, _statusVersion)]]</span>
        </div>
      </template>
    </div>
  </template>
  <template is="dom-if" if="[[_isEmpty(topByDocs)]]">
    <div class="chart-empty">No documents found in this period</div>
  </template>
</nuxeo-card>
```

Slot 5 differs only in: subhead "by storage added", header "STORAGE", it repeats
`topByStorage`, the bar uses `topByStorageMax`, and the value uses
`_formatStorage(row.value)`.

### CSS (brief)

- **`.card-half`** → `flex: 0 0 calc(50% - 16px)` → exact 50/50 in their own row.
- `.usecase-head-row` / `.usecase-row` are flex rows; the fixed column widths
  come from `.uc-rank-h/.uc-val-h/.uc-status-h` (and matching `.uc-rank/.uc-val/
  .uc-status`), while `.uc-main` is `flex:1` so the name+bar take the slack.
- `.uc-name` ellipsis-truncates long names. `.uc-bar` is the grey track;
  `.uc-bar-fill` is the blue fill whose **width** is set inline by `_barStyle`
  (note `style$=` — binds the whole style attribute). `transition: width .3s`
  animates it.
- `.uc-status` is plain centred text (per your request — no colored pills).

### Data + queries

- **Slot 4** ranks `byDocType` (count per `ecm:primaryType`).
- **Slot 5** ranks `storageByDocType` (storage sum per `ecm:primaryType`).
- **Migrated %** needs, per type: total docs (from `byDocType`, cached as
  `_docCountByType`) and migrated docs (from a per-type page-provider running
  `_typeMigratedQuery(uc.type, startDate, endDate)`).

### Build flow (observers)

**`_computeTopByDocs(byDocType)`** (observer, input the `byDocType` buckets):
1. builds `_docCountByType = {type: count}` for every type (used by the %),
2. `_topN(byDocType, 5)` → top-5 rows,
3. sets `topByDocs` and `topByDocsMax = top[0].value` (for the bar scale),
4. `_rebuildUseCaseTypes()`, then bumps `_statusVersion`.

**`_computeTopByStorage(storageByDocType)`** (observer): `_topN(...)` → sets
`topByStorage` + `topByStorageMax`, then `_rebuildUseCaseTypes()`.

**`_topN(data, n)`** — input buckets + N; sorts by value desc, takes N, shapes to
`{type: key, label: _friendlyDocType(key), value}`.

**`_friendlyDocType(type)`** — input a primary type; returns a readable name
(`"BrokerageServiceDoc" → "Brokerage Service Doc"`; camelCase gets spaced).

**`_rebuildUseCaseTypes()`** — unions the types across both top-5 lists into
`_useCaseTypes = [{type, mig:0}, ...]`, but only reassigns if the set actually
changed (avoids re-stamping the providers). Each `{type, mig}` object gives each
per-type provider its **own** `mig` field to write into.

### Per-row helper flow

- **`_inc(index)`** — input the dom-repeat 0-based `index`; returns `index+1`
  (the rank number).
- **`_barStyle(value, max)`** — inputs this row's value and the list max; returns
  `"width:NN%"` (proportional; min 4% so a tiny bar is still visible).
- **`_formatCount(row.value)`** (Slot 4) / **`_formatStorage(row.value)`**
  (Slot 5) — format the value cell.
- **`_statusPct(type, docCount, _v)`** — the migrated %:
  - **inputs:** `row.type`, the `_docCountByType` map, and `_statusVersion`
    (the third arg is only there so Polymer re-runs this when a count arrives).
  - total = `docCount[type]`; if 0 → `"0%"`. migrated = `_typeMigrated[type]`
    (defaults to 0). Returns `round(migrated/total×100) + "%"`.
  - **Always a number** — 0% until a migrated count lands, then the real value.

**`_onTypeMigratedCount(e)`** — when a per-type provider returns:
- input the event; `e.model.uc.type` is the type, `e.detail.value` the count.
- copies `_typeMigrated`, sets `map[type] = value`, reassigns it, bumps
  `_statusVersion` → every `_statusPct` re-runs and the % cells update.

### Small example (Slot 4)

`byDocType = [{key:"CustomConfigDoc",value:160},{key:"BackgroundInvestigation",
value:100},{key:"FFIORMDContractDoc",value:16},{key:"StandardFolder",value:8},
{key:"RegulatoryFolder",value:2}]`:
- `_docCountByType` = `{CustomConfigDoc:160, ...}`
- `_topN(...,5)` sorts (already sorted) → rows with labels via `_friendlyDocType`:
  `"Custom Config Doc"`, `"Background Investigation"`, `"FFIORMD Contract Doc"`,
  `"Standard Folder"`, `"Regulatory Folder"`.
- `topByDocsMax = 160`.
- Row 1: `_inc(0)=1`; `_barStyle(160,160)="width:100%"`; `_formatCount(160)="160"`;
  `_statusPct("CustomConfigDoc", map, v)` → total 160, migrated 0 → `"0%"`.
- Row 4: `_barStyle(8,160)` → 5% → min-clamped stays 5%; value "8"; "0%".
- When the per-type provider for CustomConfigDoc returns migrated=32 →
  `_typeMigrated.CustomConfigDoc=32`, `_statusVersion++` → its cell recomputes to
  `round(32/160×100)="20%"`.

---

# SLOT 6 — PLACEHOLDER

```html
<nuxeo-card class="full-width" heading="Slot 6">
  <div class="slot-placeholder">
    <iron-icon icon="icons:insert-chart"></iron-icon>
    <span class="placeholder-label">Coming soon</span>
  </div>
</nuxeo-card>
```

`.slot-placeholder` centres a faded 40px icon over "Coming soon" in a 200px-min
box. No data, no helpers — reserved space.


---

# ONE LARGE EXAMPLE — TRACING THE ENTIRE CODE

**Scenario.** Today is **2026-06-29**. The user leaves the default period
(**1month**). The repository (in this period) has:

- **298** live documents, all created in **June 2026**,
- totalling **506,388 bytes** of file content,
- of types: CustomConfigDoc ×160, BackgroundInvestigation ×100,
  FFIORMDContractDoc ×16, StandardFolder ×8, RegulatoryFolder ×14,
- with **1** document migrated (a CustomConfigDoc tagged via REST).

We trace from load to every rendered pixel.

### Step 1 — `ready()`

Runs once. Calls `_computeDates("1month")`.

### Step 2 — `_computeDates("1month")`

- `monthMap["1month"] = 1`.
- `end = 2026-06-29`; `start = 2026-05-29` (today − 1 month).
- `startDate = _toISODate(start) = "2026-05-29"`.
- `endDate = _toISODate(end) = "2026-06-29"`.

> State: `startDate="2026-05-29"`, `endDate="2026-06-29"`.

### Step 3 — assigning the dates fans out

Everything bound to `startDate`/`endDate` fires:

**(3a) Aggregations** (each computes server-side, returns buckets):
- `docsPerMonth = [{key:"2026-06-01", value:298}]` (May has 0, June 298)
- `storagePerMonth = [{key:"2026-06-01", value:506388}]`
- `byDocType = [{key:"CustomConfigDoc",value:160},{key:"BackgroundInvestigation",
  value:100},{key:"FFIORMDContractDoc",value:16},{key:"RegulatoryFolder",value:14},
  {key:"StandardFolder",value:8}]`
- `storageByDocType = [{key:"CustomConfigDoc",value:263000}, ... ]`
- `byFormat = [{key:"application/pdf",value:250},{key:"image/png",value:48}]`
- `totalStorageBytes = 506388`

**(3b) Counts** (page-provider resultsCount):
- `_totalQuery(...)` → `totalCount = 298`
- `_migratedQuery(...)` (with `/*/systemName IS NOT NULL`) → `migratedCount = 1`

**(3c) Observer `_buildMonths("2026-05-29","2026-06-29")`:**
- `_parseLocal` → start 2026-05-29, end 2026-06-29 (local).
- cursor walks May-2026 and Jun-2026 → 2 month descriptors:
  - May-26: start `2026-05-29 00:00:00`, end `2026-05-31 23:59:59`
  - Jun-26: start `2026-06-01 00:00:00`, end `2026-06-29 23:59:59`
- `_months` = those 2; `_refreshMonthly()` builds 2 zero rows for now.

### Step 4 — the dom-repeats fire their providers

**Per-month (2 months × 2 = 4 providers):**
- May migrated → 0 → `_onMigratedCount` → `e.model.set('m.migrated',0)` → refresh
- May total → 0 → `_onTotalCount` → `e.model.set('m.all',0)` → refresh
- Jun migrated → 1 → set `m.migrated=1` → refresh
- Jun total → 298 → set `m.all=298` → refresh

After the Jun writes, `_refreshMonthly()` yields:
`monthlyMigrated = [{label:"May-26",migrated:0,nonMigrated:0,all:0},
{label:"Jun-26",migrated:1,nonMigrated:297,all:298}]`.

**Observers `_computeTopByDocs(byDocType)` and `_computeTopByStorage(...)`:**
- `_docCountByType = {CustomConfigDoc:160, BackgroundInvestigation:100,
  FFIORMDContractDoc:16, RegulatoryFolder:14, StandardFolder:8}`.
- `_topN(byDocType,5)` → `topByDocs` (labels friendly), `topByDocsMax=160`.
- `_topN(storageByDocType,5)` → `topByStorage`, `topByStorageMax=263000`.
- `_rebuildUseCaseTypes()` → `_useCaseTypes = [{type:"CustomConfigDoc",mig:0},
  {type:"BackgroundInvestigation",mig:0}, ...]` (union of both lists).

**Per-type (one migrated provider per type)** run `_typeMigratedQuery`:
- CustomConfigDoc → 1 → `_onTypeMigratedCount` → `_typeMigrated={CustomConfigDoc:1}`,
  `_statusVersion++`.
- all other types → 0 → `_typeMigrated` gets those 0s, version bumps again.

### Step 5 — HEADER renders

- `_isCustom("1month")` → false → no date pickers.
- Tooltip hidden until the "i" is hovered.

### Step 6 — KPI strip renders

- `_formatCount(298)` → `"298"`
- `_formatStorage(506388)` → `506388/1024 ≈ 494.5` → `"494.5 KB"`
- `migratedCount` → `1`
- `_nonMigrated(298,1)` → `297`

> KPI: **298 · 494.5 KB · 1 · 297**.

### Step 7 — SLOT 1 Overall (default tab) renders

- `activeView="overall"` → `_isView` true for overall.
- `_monthLabels(storagePerMonth)` → `["Jun-26"]` (only one storage bucket).
- `_overallValues([{...506388}], [{...298}], 298)`:
  - `_storageUnit` → KB (divisor 1024) → storageVals `[494.52]`
  - `_docsAlignedToStorage` → `[298]`; `_reconcileDocs([298],298)` → `[298]`
  - → `[[494.52],[298]]`
- `_overallSeries` → `["Storage added in (KB)","Number of documents created"]`.
- Chart: a 494.52 storage bar + a 298 documents bar at Jun-26, one Y-axis.

### Step 8 — SLOT 2 donut renders

- `_migrationLabels()` → `["Migrated","Non-Migrated"]`
- `_migrationValues(298,1)` → `[[1,297]]`
- `_migratedPct(298,1)` → `round(0.34)=0` → center **"0%"**
- summary: Migrated 1 · Non-Migrated 297 · Total 298.

### Step 9 — SLOT 3 monthly renders

- `_monthlyEmpty(monthlyMigrated)` → false.
- `_monthlyLabels` → `["May-26","Jun-26"]`.
- `_monthlyValues` → `[[0,1],[0,297]]` → Jun shows a thin green + tall red bar.

### Step 10 — SLOTS 4 & 5 render

**Slot 4 (by docs)** rows from `topByDocs` (max 160):
| # | Use case | bar | DOCS | MIGRATED % |
|---|---|---|---|---|
| 1 | Custom Config Doc | 100% | 160 | `_statusPct` → 1/160 → **1%** |
| 2 | Background Investigation | 63% | 100 | 0/100 → **0%** |
| 3 | FFIORMD Contract Doc | 10%→ clamps small | 16 | **0%** |
| 4 | Regulatory Folder | ~9% | 14 | **0%** |
| 5 | Standard Folder | 5% | 8 | **0%** |

(Row 1's % starts at 0% for a blink, then `_onTypeMigratedCount` sets
`_typeMigrated.CustomConfigDoc=1`, bumps `_statusVersion`, and `_statusPct`
recomputes to `round(1/160×100)=1%`.)

**Slot 5 (by storage)** rows from `topByStorage` (max 263000), values via
`_formatStorage`, same `_statusPct` per type.

### Step 11 — SLOT 6

Static "Coming soon" placeholder.

### The one-sentence summary

A date drives everything: `_computeDates` sets `startDate`/`endDate`; that
re-runs the aggregations (charts) and page-provider counts (KPIs, donut,
per-month, per-type); observers assemble `_months`, `monthlyMigrated`,
`topByDocs`, `topByStorage`, and the count maps; each slot's helpers shape that
data into labels, values, units, percentages, and bar widths; and Polymer
repaints exactly the bound spots — with the CSS giving the header, the KPI strip,
the tabbed hero, the 1/3+2/3 migration row, the 50/50 use-case row, and the
responsive behaviour around it.
