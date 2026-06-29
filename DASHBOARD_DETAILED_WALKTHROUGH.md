# fid-analytical-dashboard - Detailed Technical Walkthrough

A complete, line-by-line explanation of the Nuxeo analytical dashboard.

**How to read this document**

- **CSS** is explained at a normal level (what each rule achieves visually).
- **Helper functions, the NXQL query building, and the chart wiring** are
  explained **line by line** — every statement, every guard, every transform.
- The final two sections trace a **worked example** end to end, with a
  **flowchart** of the whole reactive pipeline.

This dashboard is a **Polymer 2 Web Component** for Nuxeo Web UI (LTS 2023),
designed to work at **billion-document scale**.

---

# PART A — STRUCTURE AND CSS (overview level)

## A1. The two-file setup

```html
<link rel="import" href="fid-dashboard-styles.html">
```

Pulls in the style module. Then inside the template:

```html
<style include="iron-flex iron-flex-alignment nuxeo-styles fid-dashboard-styles"></style>
```

`include=` imports style modules by their `dom-module` id:

- `iron-flex`, `iron-flex-alignment` — Nuxeo flexbox helpers
- `nuxeo-styles` — global Nuxeo theme variables
- `fid-dashboard-styles` — your CSS module (the second file)

The second file wraps all CSS in `<dom-module id="fid-dashboard-styles">`. The id
is the entire link between the two files, which is why **both must be in the
Studio bundle**.

## A2. Page shell

```css
.page, .content { @apply --layout-vertical; }       /* stack vertically */
.dashboard-container {
  padding: 16px 24px 32px 24px;
  background: var(--nuxeo-page-background, #f0f2f5); /* theme grey backdrop */
  min-height: 100vh;                                 /* fill the screen */
  box-sizing: border-box;                            /* padding inside width */
}
```

## A3. Header card

```css
.header-slot { display:flex; justify-content:space-between; align-items:center; }
```

`justify-content:space-between` pushes the **title left, controls right**.

- `.header-left` — `flex:1` so it absorbs free space; `min-width:0` lets the
  title truncate with `text-overflow:ellipsis`.
- `.header-divider` — a 1px vertical line; `flex-shrink:0` keeps it from
  collapsing.
- `.header-controls` — holds the dropdown + date pickers, bottom-aligned.

## A4. KPI strip + the card grid

```css
.flex-layout { display:flex; flex-wrap:wrap; margin:16px -8px 0 -8px; }
.flex-layout nuxeo-card { flex: 1 0 calc(33% - 16px); margin: 0 8px 16px 8px; }
.flex-layout nuxeo-card.full-width { flex: 1 0 calc(100% - 16px); }
```

This one rule controls the **3-per-row grid**. A card with `full-width` spans the
whole row (KPI strip + hero); cards without it take a third (migration +
placeholders). The negative `-8px` margins cancel the card margins so cards sit
flush to the edges.

`.kpi-strip` lays the cells in a row; each `.kpi-cell` takes equal width with a
thin right divider (`:last-child` removes the final one).

## A5. Hero tabs and chart box

```css
.view-tab { padding:6px 14px; border-radius:16px; cursor:pointer; }
.view-tab.active { background: var(--nuxeo-primary-color,#1a73e8); color:white; }
.chart-box { position:relative; width:100%; height:380px; }
chart-line, chart-bar, chart-pie { width:100% !important; height:100% !important; }
```

The pills switch colour when `.active`. **`.chart-box` gives the chart a fixed
380px height** — Chart.js needs a fixed-height parent (with
`maintainAspectRatio:false`) or it collapses. This is the single most important
chart-layout rule.

## A6. Donut and placeholders

```css
.donut-wrap { position:relative; height:240px; }
.donut-center { position:absolute; top:0;left:0;right:0;bottom:0;
                display:flex; align-items:center; justify-content:center;
                pointer-events:none; }
```

`.donut-wrap` is `relative` so `.donut-center` can be `absolute` inside it,
stretched to all four edges and centered — that floats the "X% MIGRATED" text in
the donut hole. `pointer-events:none` lets hover/tooltips reach the chart behind.

## A7. Responsive

```css
@media (max-width:1100px){ .flex-layout nuxeo-card { flex:1 0 calc(50% - 16px); } }
@media (max-width:768px) { .flex-layout nuxeo-card { flex:1 0 calc(100% - 16px); } }
```

Cards go 3 → 2 → 1 per row as the screen narrows; on phones the header stacks and
the divider hides.

---

# PART B — THE DATA ELEMENTS (HTML that fetches)

These elements are invisible. They fetch numbers and write them into properties.

## B1. Aggregation queries (charts) — scale-safe

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

Line by line:

- `start-date="[[startDate]]"` — **one-way** binding; the element reads
  `startDate`.
- `end-date="[[_extendEndDate(endDate)]]"` — one-way, but passed through the
  helper `_extendEndDate` first (covered in C7) so the end of *today* is
  included.
- `with-date-intervals="month"` — tells Elasticsearch to bucket results **per
  month** (a date histogram).
- `date-field="dc:created"` — which date field to bucket on (creation date).
- `data="{{docsPerMonth}}"` — **two-way** binding; the element writes its result
  array back into the `docsPerMonth` property.
- `index="[[index]]"` — which ES index to query.

There are five of these: docs/month, storage/month, by doc type, by format, total
storage. They use `grouped-by` or `metrics` instead of `with-date-intervals`
where appropriate, but the binding pattern is identical.

**Why this scales:** these are ES aggregations. ES computes them across the whole
index and returns only the buckets (a handful of rows), never the documents.

## B2. Count queries (migration) — scale-safe

```html
<nuxeo-page-provider
  auto
  page-size="1"
  query="[[_totalQuery(startDate, endDate)]]"
  current-page="{{_ignoredTotalPage}}"
  results-count="{{totalCount}}">
</nuxeo-page-provider>
```

Line by line:

- `auto` — fire the query automatically whenever its bound inputs change.
- `page-size="1"` — fetch at most **1 document**. We do not want documents, only
  the count, so we keep the page tiny.
- `query="[[_totalQuery(startDate, endDate)]]"` — the **full NXQL string** built
  by `_totalQuery` (covered in C8). No `?` placeholders.
- `current-page="{{_ignoredTotalPage}}"` — the returned page of documents. We
  must bind it (the element requires it) but we ignore the value.
- `results-count="{{totalCount}}"` — **this is the count we want.** The provider
  writes the total number of matching documents into `totalCount`.

**Two hard-won details:**

1. The attribute is **`results-count`** (property `resultsCount`), NOT
   `number-of-results`. The wrong name leaves the count silently at 0.
2. We build the **whole query string** rather than using `?` placeholders. Unbound
   `?` placeholders reached the server and caused
   `Lexical Error: Illegal character <?>`.

The migrated provider is identical but uses `_migratedQuery` and writes into
`migratedCount`.

---

# PART C — THE SCRIPT, LINE BY LINE

## C1. Element registration and properties

```javascript
Polymer({
  is: 'fid-analytical-dashboard',
  behaviors: [Nuxeo.LayoutBehavior],
```

- `is:` — the custom element tag name; must match the `dom-module` id and the
  Nuxeo route.
- `behaviors: [Nuxeo.LayoutBehavior]` — mixes in Nuxeo's layout behavior (shared
  helpers Web UI elements expect).

```javascript
selectedPeriod: { type: String, value: '6months', observer: '_onPeriodChanged' },
```

- `type: String` — the property holds a string.
- `value: '6months'` — its default; the dashboard opens on a 6-month range.
- `observer: '_onPeriodChanged'` — whenever `selectedPeriod` changes, Polymer
  calls `_onPeriodChanged` automatically.

```javascript
startDate: { type: String, notify: true },
endDate:   { type: String, notify: true },
```

- `notify: true` — the property fires a `*-changed` event when it changes, so
  bound elements (the data elements) react.

```javascript
docsPerMonth: { type: Array, value: function() { return []; } },
```

- `value: function() { return []; }` — array/object defaults **must** be a
  function (so each element instance gets its own array, not a shared one).

```javascript
totalCount:    { type: Number, value: 0 },
migratedCount: { type: Number, value: 0 },
```

- Default `0` so the KPI/donut render safely before the counts arrive.

```javascript
_ignoredTotalPage:    { type: Array },
_ignoredMigratedPage: { type: Array }
```

- Required bind targets for each page-provider's `current-page`; their values are
  never read.

## C2. ready()

```javascript
ready: function() {
  this._computeDates(this.selectedPeriod);
},
```

- `ready` runs **once**, after the element is initialised.
- It calls `_computeDates` with the default `'6months'`, so `startDate` /
  `endDate` are populated immediately on load — which in turn triggers every data
  element to fetch.

## C3. _isCustom

```javascript
_isCustom: function(period) {
  return period === 'custom';
},
```

- Returns `true` only when the period is the literal string `'custom'`.
- Used by the `dom-if` that shows the manual date pickers.

## C4. _onPeriodChanged

```javascript
_onPeriodChanged: function(newVal) {
  if (!newVal || newVal === 'custom') return;
  this._computeDates(newVal);
},
```

- `newVal` — the newly selected period (Polymer passes the new value to an
  observer).
- `if (!newVal || newVal === 'custom') return;` — two guards:
  - `!newVal` — if it is empty/undefined (can happen during init), do nothing.
  - `newVal === 'custom'` — for custom, the manual date pickers drive the dates,
    so we do **not** compute from a month count; return early.
- Otherwise call `_computeDates(newVal)` to derive the date range.

## C5. _computeDates (line by line)

```javascript
_computeDates: function(period) {
  var monthMap = {
    '1month': 1, '2months': 2, '3months': 3, '4months': 4,
    '6months': 6, '8months': 8, '10months': 10, '1year': 12
  };
  var months = monthMap[period];
  if (!months) return;

  var end   = new Date();
  var start = new Date();
  start.setMonth(start.getMonth() - months);

  this.startDate = this._toISODate(start);
  this.endDate   = this._toISODate(end);
},
```

- `var monthMap = {...}` — maps each period option to a **number of months**.
- `var months = monthMap[period];` — looks up how many months this period means.
- `if (!months) return;` — if the period is not in the map (e.g. `'custom'` or a
  typo), stop; nothing to compute.
- `var end = new Date();` — `end` is **right now** (today).
- `var start = new Date();` — `start` also starts as today...
- `start.setMonth(start.getMonth() - months);` — ...then we move it **back N
  months**. `setMonth` handles year rollover automatically (e.g. Jan − 2 months →
  previous November).
- `this.startDate = this._toISODate(start);` — store the start as a
  `"YYYY-MM-DD"` string (see C6). Assigning it triggers all bindings on
  `startDate`.
- `this.endDate = this._toISODate(end);` — same for the end.

## C6. _toISODate

```javascript
_toISODate: function(date) {
  return date.toISOString().split('T')[0];
},
```

- `date.toISOString()` → e.g. `"2026-06-29T08:00:00.000Z"`.
- `.split('T')` → `["2026-06-29", "08:00:00.000Z"]`.
- `[0]` → `"2026-06-29"`. So the function returns just the date part.

## C7. _extendEndDate (line by line)

```javascript
_extendEndDate: function(date) {
  if (!date) return date;
  var d = new Date(date);
  d.setDate(d.getDate() + 1);
  d.setMilliseconds(d.getMilliseconds() - 1);
  return d.toISOString().split('T')[0];
},
```

- `if (!date) return date;` — if no date yet, return it unchanged (guards against
  `undefined` during init).
- `var d = new Date(date);` — parse the `"YYYY-MM-DD"` string into a Date object
  (midnight at the start of that day).
- `d.setDate(d.getDate() + 1);` — move to the **next** day (midnight tomorrow).
- `d.setMilliseconds(d.getMilliseconds() - 1);` — step back **1 millisecond**, so
  we land on `23:59:59.999` of the **original** day.
- `return d.toISOString().split('T')[0];` — return the date part. (For the
  aggregation elements this keeps the end inclusive of the whole day.)

## C8. The NXQL query builders (line by line)

These build the **complete** query strings for the page providers — no `?`
placeholders.

### _totalQuery

```javascript
_totalQuery: function(startDate, endDate) {
  var range = this._dateRangeClause(startDate, endDate);
  if (!range) return '';
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " + range;
},
```

- `var range = this._dateRangeClause(startDate, endDate);` — build the date part
  of the WHERE clause (see below).
- `if (!range) return '';` — if dates are not ready, `range` is `''`; return an
  empty query. An empty query means the page provider's `auto` does **not** fire
  (this is how we avoid firing a half-built query).
- `return "SELECT * FROM Document WHERE ecm:isVersion = 0 AND ecm:isTrashed = 0 " + range;`
  — the full query:
  - `ecm:isVersion = 0` — exclude version snapshots (count live docs only).
  - `ecm:isTrashed = 0` — exclude trashed docs.
  - `+ range` — append the date filter.

### _migratedQuery

```javascript
_migratedQuery: function(startDate, endDate) {
  var range = this._dateRangeClause(startDate, endDate);
  if (!range) return '';
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " + range +
         " AND fid_migration:migrationHistory/*1/systemName IS NOT NULL";
},
```

- Identical to `_totalQuery`, **plus** one extra condition:
  - `fid_migration:migrationHistory/*1/systemName IS NOT NULL` — the document has
    **at least one** entry in the multivalued complex field `migrationHistory`
    whose `systemName` is set. `/*1/` is NXQL's correlated-list syntax meaning
    "any element of the list." Empty arrays do not match, so this counts exactly
    the migrated documents.

### _dateRangeClause

```javascript
_dateRangeClause: function(startDate, endDate) {
  if (!startDate || !endDate) return '';
  var start = this._toTimestamp(startDate, false);
  var end   = this._toTimestamp(endDate, true);
  return "AND dc:created BETWEEN TIMESTAMP '" + start +
         "' AND TIMESTAMP '" + end + "'";
},
```

- `if (!startDate || !endDate) return '';` — if either date is missing, return
  empty (this is what makes `_totalQuery`/`_migratedQuery` return `''` and keeps
  the provider from firing).
- `var start = this._toTimestamp(startDate, false);` — format the start as a
  timestamp at `00:00:00`.
- `var end = this._toTimestamp(endDate, true);` — format the end as a timestamp at
  `23:59:59` (the whole final day included).
- `return "AND dc:created BETWEEN TIMESTAMP '...' AND TIMESTAMP '...'";` — the
  NXQL date filter. NXQL matches date fields against **timestamps**, and the
  `TIMESTAMP '...'` literal is the correct typed form (bare date strings or `?`
  placeholders both failed earlier).

### _toTimestamp

```javascript
_toTimestamp: function(date, isEnd) {
  if (!date) return date;
  var day = String(date).split('T')[0];
  return day + (isEnd ? ' 23:59:59' : ' 00:00:00');
},
```

- `if (!date) return date;` — guard against missing input.
- `var day = String(date).split('T')[0];` — take just the `"YYYY-MM-DD"` part
  (in case a full ISO string was passed).
- `return day + (isEnd ? ' 23:59:59' : ' 00:00:00');` — append the time:
  end-of-day for the end bound, start-of-day for the start bound. Result e.g.
  `"2026-06-29 23:59:59"`.

## C9. Tab logic (line by line)

```javascript
_isView: function(active, view) {
  return active === view;
},
```

- Returns `true` when the currently active view equals the one asked about. Each
  chart's `dom-if` calls this so only the active chart is rendered.

```javascript
_tabClass: function(active, view) {
  return active === view ? 'view-tab active' : 'view-tab';
},
```

- Returns the CSS class string for a tab pill: adds `active` (blue highlight) when
  this tab is the selected one, otherwise just `view-tab`.

```javascript
_selectOverall: function() { this.activeView = 'overall'; },
_selectDocType: function() { this.activeView = 'docType'; },
_selectFormat:  function() { this.activeView = 'format'; },
_selectSize:    function() { this.activeView = 'size'; },
```

- Each click handler sets `activeView`. Because the chart `dom-if`s and the tab
  classes are bound to `activeView`, changing it instantly swaps the visible chart
  and the highlighted pill.

## C10. Generic chart helpers (line by line)

ES returns buckets like `[{ key: "...", value: N }, ...]`.

```javascript
_isEmpty: function(data) {
  return !data || !data.length;
},
```

- `true` if `data` is missing/null OR has length 0. Drives the "No documents
  found" empty states.

```javascript
_labels: function(data) {
  if (!data || !data.length) return [];
  return data.map(function(entry) { return entry.key; });
},
```

- `if (!data || !data.length) return [];` — empty input → empty label array.
- `data.map(... entry.key ...)` — pull the `key` from each bucket; these become
  the axis/pie labels. e.g. `["File","Note","Picture"]`.

```javascript
_values: function(data) {
  if (!data || !data.length) return [[]];
  return [data.map(function(entry) { return entry.value; })];
},
```

- `if (!data || !data.length) return [[]];` — empty input → `[[]]` (an array
  containing one empty series; chart-elements expects an array of series).
- `return [ data.map(... entry.value ...) ];` — pull each bucket's `value` into an
  inner array, then wrap it in an **outer** array. So `[[5, 8, 3]]` = one series
  of three points.

```javascript
_monthLabels: function(data) {
  if (!data || !data.length) return [];
  var self = this;
  return data.map(function(entry) { return self._toMonthYear(entry.key); });
},
```

- `var self = this;` — capture `this` so the inner function (which has its own
  `this`) can still call `self._toMonthYear`.
- Map each bucket key through `_toMonthYear` to get friendly month labels.

```javascript
_toMonthYear: function(key) {
  if (!key) return '';
  var months = ['Jan','Feb',...,'Dec'];
  var parts = String(key).split('-');
  if (parts.length < 2) return key;
  var year = parts[0];
  var monthIdx = parseInt(parts[1], 10) - 1;
  if (monthIdx < 0 || monthIdx > 11) return key;
  return months[monthIdx] + '-' + year.slice(-2);
},
```

- `if (!key) return '';` — guard empty key.
- `var months = [...]` — month abbreviations indexed 0–11.
- `var parts = String(key).split('-');` — split `"2026-03-01"` into
  `["2026","03","01"]`.
- `if (parts.length < 2) return key;` — if it does not look like a date, return it
  unchanged.
- `var year = parts[0];` — `"2026"`.
- `var monthIdx = parseInt(parts[1], 10) - 1;` — `"03"` → number `3` → index `2`
  (arrays are 0-based). `10` is the radix (base-10) so leading zeros do not cause
  octal parsing.
- `if (monthIdx < 0 || monthIdx > 11) return key;` — bounds check.
- `return months[monthIdx] + '-' + year.slice(-2);` — `"Mar" + "-" + "26"` →
  `"Mar-26"`. `slice(-2)` takes the last two digits of the year.

## C11. Overall combined-chart helpers (line by line)

```javascript
_docsAlignedToStorage: function(storage, docs) {
  var docMap = {};
  (docs || []).forEach(function(entry) { docMap[entry.key] = entry.value; });
  return (storage || []).map(function(entry) {
    return docMap[entry.key] || 0;
  });
},
```

- `var docMap = {};` — build a lookup of documents keyed by month.
- `(docs || []).forEach(... docMap[entry.key] = entry.value ...)` — fill the
  lookup: `{ "2026-03-01": 40, ... }`. The `docs || []` guards null.
- `return (storage || []).map(... docMap[entry.key] || 0 ...)` — walk the
  **storage** months (the x-axis we anchor on) and, for each month, pull the
  matching document count, or `0` if that month has no documents. This guarantees
  both datasets line up on the same months.

```javascript
_overallValues: function(storage, docs) {
  if (!storage || !storage.length) return [[]];
  var unit = this._storageUnit(storage);
  var storageVals = storage.map(function(entry) {
    var v = (entry.value || 0) / unit.divisor;
    return Math.round(v * 100) / 100;
  });
  var docVals = this._docsAlignedToStorage(storage, docs);
  return [storageVals, docVals];
},
```

- `if (!storage || !storage.length) return [[]];` — no storage data → one empty
  series.
- `var unit = this._storageUnit(storage);` — choose KB/MB/GB/TB (see C12).
- `storageVals = storage.map(...)`:
  - `var v = (entry.value || 0) / unit.divisor;` — bytes → chosen unit.
  - `return Math.round(v * 100) / 100;` — round to **2 decimals** (×100, round,
    ÷100).
- `var docVals = this._docsAlignedToStorage(storage, docs);` — the aligned
  document counts.
- `return [storageVals, docVals];` — **two series**: storage first, documents
  second. Both render as grouped bars on the shared axis.

```javascript
_overallSeries: function(storage) {
  return ['Storage (' + this._storageUnit(storage).name + ')', 'Documents'];
},
```

- Returns the two legend labels. The storage label includes the chosen unit, e.g.
  `["Storage (KB)", "Documents"]`.

```javascript
_overallOptions: function(storage) {
  return {
    responsive: true,
    maintainAspectRatio: false,
    animation: false,
    legend: { display: true, position: 'top', labels: {...} },
    scales: {
      xAxes: [{ gridLines:{...}, ticks:{...} }],
      yAxes: [{ gridLines:{...}, ticks:{ beginAtZero:true, ... } }]
    },
    tooltips: { mode: 'index', intersect: false }
  };
},
```

- Returns a **Chart.js options object** (not a string, because it is complex).
- `responsive:true` + `maintainAspectRatio:false` — let the chart fill the fixed
  380px `.chart-box` instead of forcing its own aspect ratio.
- `animation:false` — no animation (snappy, and avoids redraw flicker).
- `legend` — top-positioned legend with small swatches.
- `scales.xAxes` — one x-axis, gridlines off, labels rotated up to 30°.
- `scales.yAxes` — **one** y-axis, `beginAtZero:true`. (We deliberately use a
  single shared axis; a second, unused axis previously fell back to a phantom 0–1
  scale.)
- `tooltips: { mode:'index', intersect:false }` — hovering a month shows **both**
  series' values at once.

## C12. Storage helpers (line by line)

```javascript
_storageMax: function(data) {
  if (!data || !data.length) return 0;
  return data.reduce(function(max, entry) {
    var v = entry.value || 0;
    return v > max ? v : max;
  }, 0);
},
```

- Finds the **largest** monthly byte value.
- `reduce(..., 0)` starts the running `max` at 0 and keeps the larger of
  `max`/current each step.

```javascript
_storageUnit: function(data) {
  var max = this._storageMax(data);
  if (max >= 1099511627776) return { name: 'TB', divisor: 1099511627776 };
  if (max >= 1073741824)    return { name: 'GB', divisor: 1073741824 };
  if (max >= 1048576)       return { name: 'MB', divisor: 1048576 };
  return { name: 'KB', divisor: 1024 };
},
```

- Picks the display unit from the max value, comparing against byte thresholds:
  - `1099511627776` = 1024⁴ = 1 TB
  - `1073741824` = 1024³ = 1 GB
  - `1048576` = 1024² = 1 MB
  - else KB (1024).
- Returns `{name, divisor}`. This is why tiny test data shows as KB while real
  data shows as GB/TB — the unit auto-scales.

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

- Same conversion as in `_overallValues`, but wrapped as a **single** series
  `[[...]]` (used by the standalone "By Size" storage bar).

```javascript
_storageSeries: function(data) {
  return ['Storage (' + this._storageUnit(data).name + ')'];
},
```

- The single legend label, reflecting the chosen unit.

## C13. Format/mime helpers (line by line)

```javascript
_typeLabels: function(data) {
  if (!data || !data.length) return [];
  var self = this;
  return data.map(function(entry) { return self._friendlyType(entry.key); });
},
```

- Maps each mime-type bucket key through `_friendlyType` for nicer pie labels.

```javascript
_friendlyType: function(mime) {
  if (!mime) return 'Unknown';
  var map = { 'application/pdf':'PDF', 'image/png':'PNG Image', ... };
  if (map[mime]) return map[mime];
  var parts = mime.split('/');
  return parts.length > 1 ? parts[1].toUpperCase() : mime;
},
```

- `if (!mime) return 'Unknown';` — guard empty.
- `var map = {...}` — known mime types → friendly names.
- `if (map[mime]) return map[mime];` — use the friendly name if known.
- `var parts = mime.split('/');` — otherwise split e.g.
  `"application/x-custom"` → `["application","x-custom"]`.
- `return parts.length > 1 ? parts[1].toUpperCase() : mime;` — return the subtype
  uppercased (`"X-CUSTOM"`), or the raw string if there is no slash.

## C14. Migration helpers (line by line)

```javascript
_migrationLabels: function() {
  return ['Migrated', 'Non-Migrated'];
},
```

- The two donut slice labels.

```javascript
_nonMigrated: function(total, migrated) {
  var diff = (total || 0) - (migrated || 0);
  return diff > 0 ? diff : 0;
},
```

- `var diff = (total || 0) - (migrated || 0);` — subtract migrated from total
  (the `|| 0` guards against undefined).
- `return diff > 0 ? diff : 0;` — never return a negative (floors at 0).

```javascript
_migrationValues: function(total, migrated) {
  return [[migrated || 0, this._nonMigrated(total, migrated)]];
},
```

- Returns the donut data as one series of two slices:
  `[[migrated, nonMigrated]]`.

```javascript
_migratedPct: function(total, migrated) {
  var t = total || 0;
  if (t <= 0) return 0;
  return Math.round(((migrated || 0) / t) * 100);
},
```

- `var t = total || 0;` — total, guarded.
- `if (t <= 0) return 0;` — avoid divide-by-zero; with no documents, 0%.
- `return Math.round(((migrated || 0) / t) * 100);` — migrated ÷ total × 100,
  rounded to a whole number. This is the big "X% MIGRATED" figure.

## C15. KPI formatting (line by line)

```javascript
_formatCount: function(n) {
  n = n || 0;
  if (n >= 1000000000) return (Math.round(n / 100000000) / 10) + 'B';
  if (n >= 1000000)    return (Math.round(n / 100000) / 10) + 'M';
  if (n >= 1000)       return (Math.round(n / 100) / 10) + 'K';
  return String(n);
},
```

- `n = n || 0;` — default to 0.
- Each branch abbreviates and keeps **one decimal**. Example for `1500000`:
  `Math.round(1500000/100000)=15`, `15/10=1.5`, → `"1.5M"`.
- Thresholds: billion → `B`, million → `M`, thousand → `K`, else the raw number.

```javascript
_formatStorage: function(bytes) {
  bytes = bytes || 0;
  if (bytes >= 1125899906842624) return (Math.round(bytes / 1125899906842624 * 10) / 10) + ' PB';
  if (bytes >= 1099511627776)    return (Math.round(bytes / 1099511627776 * 10) / 10) + ' TB';
  if (bytes >= 1073741824)       return (Math.round(bytes / 1073741824 * 10) / 10) + ' GB';
  if (bytes >= 1048576)          return (Math.round(bytes / 1048576 * 10) / 10) + ' MB';
  if (bytes >= 1024)             return (Math.round(bytes / 1024 * 10) / 10) + ' KB';
  return bytes + ' B';
},
```

- Same idea for bytes, with one decimal, scaling PB/TB/GB/MB/KB/B. Thresholds are
  the powers of 1024 (`1024⁵`=PB, `1024⁴`=TB, etc.). Example for `471,654` bytes:
  `≥1048576`? no; `≥1024`? yes → `471654/1024≈460.6` → `"460.7 KB"`.

---

# PART D — WORKED EXAMPLE (end to end)

**Scenario:** the page loads with the default period `6months`, today is
`2026-06-29`, the repo has 296 documents created in the last 6 months, none
migrated, totalling 471,654 bytes of file content.

**Step 1 — element initialises.**
`ready()` runs → `this._computeDates('6months')`.

**Step 2 — compute dates.**
In `_computeDates('6months')`:
- `monthMap['6months']` = 6 → `months = 6`.
- `end = 2026-06-29`. `start = 2026-06-29`, then `setMonth(month-6)` →
  `2025-12-29`.
- `this.startDate = "2025-12-29"`, `this.endDate = "2026-06-29"`.

**Step 3 — bindings fire.**
Setting `startDate`/`endDate` notifies every bound element:

- The **aggregation** elements run ES date-histogram / sum / grouped queries and
  fill `docsPerMonth`, `storagePerMonth`, `byDocType`, `byFormat`,
  `totalStorageBytes`.
- The **page providers** evaluate their `query` bindings:
  - `_totalQuery("2025-12-29","2026-06-29")`:
    - `_dateRangeClause` → `_toTimestamp` →
      `AND dc:created BETWEEN TIMESTAMP '2025-12-29 00:00:00' AND TIMESTAMP '2026-06-29 23:59:59'`.
    - Full query:
      `SELECT * FROM Document WHERE ecm:isVersion = 0 AND ecm:isTrashed = 0 AND dc:created BETWEEN TIMESTAMP '2025-12-29 00:00:00' AND TIMESTAMP '2026-06-29 23:59:59'`.
    - Provider fires (query non-empty) → `results-count` writes
      `totalCount = 296`.
  - `_migratedQuery(...)` adds
    `AND fid_migration:migrationHistory/*1/systemName IS NOT NULL` → matches 0 →
    `migratedCount = 0`.

**Step 4 — helpers reshape for display.**

- KPI "New documents": `_formatCount(296)` → `"296"`.
- KPI "Storage added": `_formatStorage(471654)` → `"460.7 KB"`.
- KPI "From migrated": `migratedCount` → `0`.
- KPI "From non-migrated": `_nonMigrated(296, 0)` → `296`.
- Donut center: `_migratedPct(296, 0)` → `0` → `"0%"`.
- Donut slices: `_migrationValues(296, 0)` → `[[0, 296]]` → all red
  (non-migrated).

**Step 5 — charts render (Overall tab is active by default).**

- x-axis labels: `_monthLabels(storagePerMonth)` →
  `["Dec-25","Jan-26",...,"Jun-26"]`.
- values: `_overallValues(storagePerMonth, docsPerMonth)` →
  `[[storage per month in KB], [documents per month]]`.
- series: `_overallSeries(...)` → `["Storage (KB)", "Documents"]`.
- options: `_overallOptions(...)` → single shared y-axis, grouped bars.

**Step 6 — user clicks "By Doc Type".**
`_selectDocType()` sets `activeView = 'docType'` → the Overall `dom-if` removes
its chart, the docType `dom-if` (`_isView(activeView,'docType')` now true) stamps
its pie, fed by `_labels(byDocType)` and `_values(byDocType)`.

**Step 7 — user changes period to "1month".**
`selectedPeriod` changes → observer `_onPeriodChanged('1month')` → not custom →
`_computeDates('1month')` → new `startDate`/`endDate` → **every** data element
refetches → all KPIs, counts, and charts update automatically.

---

# PART E — FLOWCHART (the reactive pipeline)

```
                         ┌─────────────────────────┐
                         │   Element loads (ready) │
                         └────────────┬────────────┘
                                      │ _computeDates('6months')
                                      ▼
                         ┌─────────────────────────┐
                         │  startDate / endDate set │
                         └────────────┬────────────┘
                                      │ (bindings notify)
              ┌───────────────────────┼───────────────────────────┐
              ▼                                                   ▼
  ┌───────────────────────┐                       ┌───────────────────────────┐
  │  nuxeo-repository-data │                       │   nuxeo-page-provider     │
  │  (5 aggregations)      │                       │   (2 counts)              │
  │                        │                       │  query = _totalQuery(...)  │
  │  ES date-histogram /   │                       │          _migratedQuery(.)│
  │  sum / grouped-by      │                       │  built via                │
  │                        │                       │  _dateRangeClause →        │
  │                        │                       │  _toTimestamp              │
  └───────────┬────────────┘                       └─────────────┬─────────────┘
              │ writes arrays                                     │ writes counts
              ▼                                                   ▼
  docsPerMonth, storagePerMonth,                       totalCount, migratedCount
  byDocType, byFormat, totalStorageBytes
              │                                                   │
              └───────────────────────┬───────────────────────────┘
                                      │ (helpers reshape)
              ┌───────────────────────┼───────────────────────────┐
              ▼                       ▼                            ▼
   _labels / _values /     _overallValues / _overallSeries   _migrationValues /
   _monthLabels /          / _overallOptions /               _migratedPct /
   _typeLabels             _storageValues / _storageSeries    _nonMigrated
              │                       │                            │
              ▼                       ▼                            ▼
   ┌───────────────────┐   ┌───────────────────────┐   ┌───────────────────────┐
   │  By Doc Type pie  │   │  Overall bars (hero)  │   │  Migration donut +    │
   │  By Format pie    │   │  By Size storage bar  │   │  KPI counts           │
   └───────────────────┘   └───────────────────────┘   └───────────────────────┘

   User actions:
     • clicking a tab  → _select*  → activeView changes → dom-if swaps the chart
     • changing period → _onPeriodChanged → _computeDates → dates change →
                         the whole pipeline above re-runs automatically
```

---

# APPENDIX — Why two query mechanisms (scale design)

- **Charts** use `nuxeo-repository-data` aggregations (`grouped-by`,
  `with-date-intervals`, `sum`). ES summarises server-side and returns only
  buckets — scales to a billion documents.
- **Counts** use `nuxeo-page-provider` with NXQL and `page-size="1"`, reading
  `results-count`. A filtered count returns one number without fetching documents
  — also scales.
- We do **not** count migrated docs with a term aggregation on `systemName`: at
  scale that needs one bucket per source system and, capped by `group-limit`,
  would undercount. A filtered `results-count` is exact.
- We avoid the `where` attribute on `nuxeo-repository-data`: on this build it
  crashes on ranges / `LIKE` / `IS NOT NULL`. NXQL inside a page provider handles
  the complex-field `IS NOT NULL` predicate correctly.

**Two bugs worth remembering:**

1. **`?` placeholders** in a page-provider query reached the server
   unsubstituted → `Lexical Error: Illegal character <?>`. Fix: build the whole
   query string with dates inlined as `TIMESTAMP '...'`.
2. **The count attribute is `results-count`**, not `number-of-results`. The wrong
   name silently leaves the count at 0.
