# Top 5 Use Cases (Slots 4 & 5) — Detailed Flow

This explains the two **Top 5 Use Cases** cards: Slot 4 ranks document types by
**document count**, Slot 5 by **storage added**. Each row shows a rank, a name, a
progress bar, the value, and a **migrated %**.

Same from-scratch style as before: every helper, line by line, with the
**intermediate result after each step**, and exactly **when each helper is called
and what calls the next one**.

A quick recap of the two ideas you already know:
- A **helper** is a named recipe: takes input, returns output.
- **`[[ ]]`** reads a value onto the screen; **`{{ }}`** lets the screen write
  back; a **`dom-repeat`** stamps its content once per item in a list, exposing
  the current item under a name (here `uc` or `row`).

---

# PART 1 — WHAT THESE SLOTS NEED TO SHOW

Each card is a small table. Slot 4 (by document volume) looks like:

```
#   USE CASE                    DOCS    MIGRATED %
1   Custom Config Doc  ▓▓▓▓▓▓▓  160     1%
2   Background Investigation ▓▓▓ 100    0%
3   FFIORMD Contract Doc ▓      16      0%
4   Regulatory Folder  ▓        14      0%
5   Standard Folder    ▓        8       0%
```

To build this the code needs three things:
1. **The top-5 ranked list** (type, friendly name, value) → property `topByDocs`
   (and `topByStorage` for Slot 5).
2. **The bar scale** — the biggest value, so bars can be drawn as a % of it →
   `topByDocsMax` / `topByStorageMax`.
3. **The migrated %** per type → needs the **total** docs of that type (from an
   aggregation we already have) and the **migrated** docs of that type (from a
   small per-type query).

---

# PART 2 — WHERE THE DATA COMES FROM

Two Elasticsearch aggregations (fetched at the top of the page) feed these slots:

- **`byDocType`** — documents grouped by `ecm:primaryType`, i.e. a count per type.
  Shape: `[{key:"CustomConfigDoc", value:160}, {key:"Note", value:100}, ...]`.
- **`storageByDocType`** — same grouping but summing file sizes. Shape:
  `[{key:"CustomConfigDoc", value:263000}, ...]` (bytes).

An **observer** links each aggregation to a builder:

```javascript
observers: [
  '_buildMonths(startDate, endDate)',
  '_computeTopByDocs(byDocType)',          // runs when byDocType changes
  '_computeTopByStorage(storageByDocType)' // runs when storageByDocType changes
],
```

"Observer" = "watch this property; when it changes, run this function." So the
moment `byDocType` arrives, `_computeTopByDocs` runs automatically.

---

# PART 3 — THE HELPERS, LINE BY LINE (with intermediate results)

We'll use this running data for Slot 4:

```
byDocType = [
  {key:"CustomConfigDoc",        value:160},
  {key:"BackgroundInvestigation",value:100},
  {key:"FFIORMDContractDoc",     value:16},
  {key:"RegulatoryFolder",       value:14},
  {key:"StandardFolder",         value:8}
]
```

## 3.1 `_computeTopByDocs(data)` — the entry point for Slot 4

Called automatically when `byDocType` changes. **Input `data`** = the `byDocType`
array above.

```javascript
_computeTopByDocs: function(data) {
  var map = {};                                   // (1) empty lookup
  (data || []).forEach(function(entry) {          // (2) loop each bucket
    if (entry && entry.key) map[entry.key] = entry.value || 0;
  });
  this._docCountByType = map;                      // (3) store the lookup
  var top = this._topN(data, 5);                   // (4) rank to top-5
  this.topByDocs = top;                            // (5) store the list
  this.topByDocsMax = top.length ? top[0].value : 0; // (6) biggest value
  this._rebuildUseCaseTypes();                     // (7) refresh type list
  this._statusVersion++;                           // (8) nudge the % helpers
}
```

Line by line, with the result after each:

1. `var map = {}` — start an empty **lookup object** (a set of key→value pairs).
2. `.forEach(...)` — go through each bucket and copy `key → value` into `map`.
   After the loop:
   ```
   map = { CustomConfigDoc:160, BackgroundInvestigation:100,
           FFIORMDContractDoc:16, RegulatoryFolder:14, StandardFolder:8 }
   ```
3. `this._docCountByType = map` — save that lookup in a property. **This is the
   "total docs per type" that the migrated % will divide by later.**
4. `var top = this._topN(data, 5)` — call `_topN` to sort and take the top 5
   (details in 3.3). Returns a shaped list (see below).
5. `this.topByDocs = top` — store it; the Slot 4 table is wired to this.
6. `this.topByDocsMax = top[0].value` — the first item is the biggest (list is
   sorted), so `topByDocsMax = 160`. Bars are drawn relative to this.
7. `this._rebuildUseCaseTypes()` — refresh the list of distinct types that need a
   migrated-count query (details in 3.5).
8. `this._statusVersion++` — increases a counter. Its only job: force the % cells
   to recompute now that `_docCountByType` is known. (Explained in 3.9.)

**Intermediate result** after `_computeTopByDocs`:
```
_docCountByType = {CustomConfigDoc:160, BackgroundInvestigation:100,
                   FFIORMDContractDoc:16, RegulatoryFolder:14, StandardFolder:8}
topByDocs   = [ {type:"CustomConfigDoc",label:"Custom Config Doc",value:160}, ... ]
topByDocsMax = 160
_statusVersion = (previous + 1)
```

## 3.2 `_computeTopByStorage(data)` — the entry point for Slot 5

Called when `storageByDocType` changes. Simpler — it doesn't build a count lookup
(the % reuses the doc-count one from Slot 4).

```javascript
_computeTopByStorage: function(data) {
  var top = this._topN(data, 5);                   // rank by storage value
  this.topByStorage = top;                         // store list for Slot 5
  this.topByStorageMax = top.length ? top[0].value : 0;
  this._rebuildUseCaseTypes();                     // refresh type list
}
```

If `storageByDocType = [{key:"CustomConfigDoc",value:263000}, {key:"Video",
value:120000}, ...]`, then after this: `topByStorage` is the shaped top-5 and
`topByStorageMax = 263000`.

## 3.3 `_topN(data, n)` — sort and take the top N

Called by both builders. **Inputs:** `data` (the buckets) and `n` (how many, here
5). **Output:** a shaped list `[{type, label, value}, ...]`.

```javascript
_topN: function(data, n) {
  if (!data || !data.length) return [];            // (1) empty guard
  var self = this;                                 // (2) keep a ref to "this"
  var copy = data.slice().sort(function(a, b) {    // (3) copy + sort desc
    return (b.value || 0) - (a.value || 0);
  });
  return copy.slice(0, n).map(function(entry) {    // (4) take N, reshape
    return {
      type:  entry.key,
      label: self._friendlyDocType(entry.key),
      value: entry.value || 0
    };
  });
}
```

1. If there's no data, return an empty list.
2. `var self = this` — a small trick. Inside the `.map(...)` below, the word
   `this` would change meaning, so we save the real `this` as `self` to call
   `self._friendlyDocType`.
3. `data.slice()` makes a **copy** (so we don't reorder the original), then
   `.sort(...)` orders it by `value` **largest first** (`b.value - a.value`).
4. `.slice(0, n)` keeps the first `n`; `.map(...)` turns each raw bucket
   `{key, value}` into a tidy `{type, label, value}`, where `label` is the
   readable name from `_friendlyDocType`.

**Intermediate result** for our data (already in order):
```
[
  {type:"CustomConfigDoc",        label:"Custom Config Doc",        value:160},
  {type:"BackgroundInvestigation",label:"Background Investigation", value:100},
  {type:"FFIORMDContractDoc",     label:"FFIORMD Contract Doc",     value:16},
  {type:"RegulatoryFolder",       label:"Regulatory Folder",        value:14},
  {type:"StandardFolder",         label:"Standard Folder",          value:8}
]
```

## 3.4 `_friendlyDocType(type)` — raw type → readable name

Called by `_topN` once per row. **Input:** a raw primary type string.
**Output:** a human name.

```javascript
_friendlyDocType: function(type) {
  if (!type) return 'Unknown';
  var map = { 'File':'File', 'Note':'Note', 'Picture':'Picture',
              'Video':'Video', 'Audio':'Audio',
              'BrokerageServiceDoc':'Brokerage Service Doc' };
  if (map[type]) return map[type];                 // known name?
  return String(type).replace(/([a-z])([A-Z])/g, '$1 $2'); // else space camelCase
}
```

- If the type is in the small `map`, return the nice name.
- Otherwise, the `.replace(...)` inserts a space between a lowercase letter
  followed by an uppercase one. So `"CustomConfigDoc"` → `"Custom Config Doc"`,
  `"FFIORMDContractDoc"` → `"FFIORMD Contract Doc"`.

Examples: `_friendlyDocType("StandardFolder")` → `"Standard Folder"`;
`_friendlyDocType("BrokerageServiceDoc")` → `"Brokerage Service Doc"` (from the map).

## 3.5 `_rebuildUseCaseTypes()` — the list of types that need a migrated query

Called at the end of both builders. It collects the **distinct** types shown
across *both* Slot 4 and Slot 5, so each gets exactly one migrated-count query.

```javascript
_rebuildUseCaseTypes: function() {
  var seen = {};                                   // (1) track duplicates
  var types = [];                                  // (2) result list of names
  (this.topByDocs || []).concat(this.topByStorage || []).forEach(function(r) {
    if (r && r.type && !seen[r.type]) {            // (3) first time seen?
      seen[r.type] = true; types.push(r.type);
    }
  });
  var current = (this._useCaseTypes || []).map(function(o){ return o.type; });
  if (current.join('|') === types.join('|')) return; // (4) unchanged? stop
  this._useCaseTypes = types.map(function(t){ return { type: t, mig: 0 }; }); // (5)
}
```

1–2. Start a `seen` marker set and an empty `types` list.
3. `.concat(...)` glues the two top-5 lists together, then we loop; each type is
   pushed **only the first time** we see it (dedupe). So if a type appears in both
   slots, it's counted once.
4. Compare the new type list to the current one (joined into a string). If they're
   identical, **return early** — don't rebuild (this avoids needlessly re-running
   all the queries).
5. Otherwise store `_useCaseTypes` as a list of **objects**, each `{type, mig:0}`.
   The `mig:0` field is a private slot for that type's migrated count; giving each
   type its own object means each query writes into its own place.

**Intermediate result** (types unique to our data):
```
_useCaseTypes = [
  {type:"CustomConfigDoc", mig:0}, {type:"BackgroundInvestigation", mig:0},
  {type:"FFIORMDContractDoc", mig:0}, {type:"RegulatoryFolder", mig:0},
  {type:"StandardFolder", mig:0}
]
```

## 3.6 JUMP TO THE `dom-repeat` — the per-type queries fire

`_useCaseTypes` is now filled, and on the page (before the visible UI) there is a
repeater wired to it:

```html
<template is="dom-repeat" items="[[_useCaseTypes]]" as="uc">
  <nuxeo-page-provider auto page-size="1"
     query="[[_typeMigratedQuery(uc.type, startDate, endDate)]]"
     results-count="{{uc.mig}}"
     on-results-count-changed="_onTypeMigratedCount"></nuxeo-page-provider>
</template>
```

- `items="[[_useCaseTypes]]" as="uc"` — repeat once per type; the current type
  object is **`uc`** (so `uc.type` is the name, `uc.mig` its count slot).
- One `nuxeo-page-provider` per type. Its query is built by
  `_typeMigratedQuery(uc.type, startDate, endDate)`.
- `results-count="{{uc.mig}}"` — the count is written into **this type's own**
  `mig` slot (unique per row — that's why the event fires reliably even if two
  types share the same count).
- `on-results-count-changed="_onTypeMigratedCount"` — when the count arrives, call
  the handler.

So the moment `_useCaseTypes` has 5 entries, **5 migrated-count queries fire**.

## 3.7 `_typeMigratedQuery(t, startDate, endDate)` — one type's migrated query

**Inputs:** the type name `t` (from `uc.type`), and the period dates.
**Output:** the NXQL to count migrated docs of that type in the period.

```javascript
_typeMigratedQuery: function(t, startDate, endDate) {
  var range = this._dateRangeClause(startDate, endDate); // "AND dc:created BETWEEN ..."
  if (!range || !t) return '';                            // guard: not ready
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " + range +
         " AND ecm:primaryType = '" + t + "'" +
         " AND fid_migration:migrationHistory/*/systemName IS NOT NULL";
}
```

For `t = "CustomConfigDoc"` and the period, it produces:
```
SELECT * FROM Document WHERE ecm:isVersion = 0 AND ecm:isTrashed = 0
AND dc:created BETWEEN TIMESTAMP '...' AND TIMESTAMP '...'
AND ecm:primaryType = 'CustomConfigDoc'
AND fid_migration:migrationHistory/*/systemName IS NOT NULL
```

Plain English: "count live docs of type CustomConfigDoc, created in the period,
that are migrated." The `/*/` shape is the one that works on this build.

## 3.8 `_onTypeMigratedCount(e)` — catch one type's migrated count

Runs when a per-type provider reports. **Input:** the event `e`.
- `e.model.uc` = the type object this provider belongs to (so `e.model.uc.type`
  is the type name).
- `e.detail.value` = the migrated count found.

```javascript
_onTypeMigratedCount: function(e) {
  if (!e || !e.model || !e.model.uc || !e.detail) return;  // safety
  var val = e.detail.value;                                // e.g. 1
  if (typeof val !== 'number') return;
  var type = e.model.uc.type;                              // e.g. "CustomConfigDoc"
  var map = {};                                            // copy the current map
  var src = this._typeMigrated || {};
  for (var k in src) { if (src.hasOwnProperty(k)) map[k] = src[k]; }
  map[type] = val;                                         // set this type's count
  this._typeMigrated = map;                                // store new map
  this._statusVersion++;                                   // trigger % recompute
}
```

Why copy the map instead of writing `this._typeMigrated[type] = val` directly?
Same reason as the monthly chart: assigning a **new** object (and bumping
`_statusVersion`) makes Polymer notice the change and re-run the % helper. A
direct in-place write would be silent.

**Intermediate result** after CustomConfigDoc's provider returns `1`:
```
_typeMigrated = { CustomConfigDoc: 1 }
_statusVersion = (previous + 1)
```
Other types return `0` and get added similarly: `{CustomConfigDoc:1,
BackgroundInvestigation:0, FFIORMDContractDoc:0, RegulatoryFolder:0,
StandardFolder:0}`.

## 3.9 JUMP TO THE VISIBLE TABLE — the row `dom-repeat`

Now the card renders. Slot 4's table repeats over `topByDocs`:

```html
<template is="dom-repeat" items="[[topByDocs]]" as="row">
  <div class="usecase-row">
    <span class="uc-rank">[[_inc(index)]]</span>
    <div class="uc-main">
      <span class="uc-name">[[row.label]]</span>
      <div class="uc-bar"><div class="uc-bar-fill"
           style$="[[_barStyle(row.value, topByDocsMax)]]"></div></div>
    </div>
    <span class="uc-val">[[_formatCount(row.value)]]</span>
    <span class="uc-status">[[_statusPct(row.type, _docCountByType, _statusVersion)]]</span>
  </div>
</template>
```

- `items="[[topByDocs]]" as="row"` — one row per ranked item; the current item is
  **`row`** (`row.type`, `row.label`, `row.value`). `index` is the 0-based
  position Polymer gives each repeat.
- Each cell calls a helper: `_inc(index)`, `row.label` (direct), `_barStyle(...)`,
  `_formatCount(row.value)`, `_statusPct(...)`.

## 3.10 `_inc(index)` — the rank number

**Input:** the 0-based `index`. **Output:** `index + 1`.

```javascript
_inc: function(index) { return index + 1; }
```

Row 0 → shows **1**, row 1 → **2**, … Because humans count from 1 but the computer
counts from 0.

## 3.11 `_barStyle(value, max)` — the progress-bar width

**Inputs:** this row's `value` and the list's `max` (`topByDocsMax`).
**Output:** an inline CSS width string like `"width:63%"`.

```javascript
_barStyle: function(value, max) {
  var pct = (max && max > 0) ? Math.round((value / max) * 100) : 0;
  if (pct < 4 && value > 0) pct = 4;   // keep a sliver visible for tiny values
  return 'width:' + pct + '%';
}
```

- Percentage = `value / max × 100`, rounded.
- If it rounds to under 4% but isn't zero, force 4% so a tiny bar is still
  visible.
- Returned via `style$=` (binds the whole style attribute), setting the blue
  fill's width.

**Intermediate results** (max = 160):
- Row 1: `_barStyle(160,160)` → 100% → `"width:100%"`
- Row 2: `_barStyle(100,160)` → 63% → `"width:63%"`
- Row 3: `_barStyle(16,160)`  → 10% → `"width:10%"`
- Row 4: `_barStyle(14,160)`  → 9%  → `"width:9%"`
- Row 5: `_barStyle(8,160)`   → 5%  → `"width:5%"`

(For Slot 5, the same helper is called with `topByStorageMax`.)

## 3.12 `_statusPct(type, docCount, _v)` — the migrated %

**Inputs:** `row.type`, the `_docCountByType` lookup, and `_statusVersion`. The
third argument `_v` is **never used inside** — it's only there so that when
`_statusVersion` changes (a new migrated count arrived), Polymer re-runs this
helper. That's the trick that refreshes the % when counts land.

```javascript
_statusPct: function(type, docCount, _v) {
  var total = (docCount || {})[type] || 0;      // total docs of this type
  if (total <= 0) return '0%';                  // no docs -> 0%
  var migrated = (this._typeMigrated || {})[type] || 0; // migrated (default 0)
  var pct = Math.round((migrated / total) * 100);
  return pct + '%';
}
```

- `total` comes from `_docCountByType` (the lookup built in 3.1) — so it's known
  immediately, even before any migrated query returns.
- `migrated` comes from `_typeMigrated` (filled in 3.8), **defaulting to 0**.
- So the cell **always shows a number**: `0%` until a migrated count arrives, then
  the real value.

**Intermediate results:**
- Before any migrated query returns (`_typeMigrated` empty):
  - CustomConfigDoc: total 160, migrated 0 → `"0%"`
  - all others → `"0%"`
- After CustomConfigDoc's query returns `1` (`_typeMigrated={CustomConfigDoc:1}`,
  `_statusVersion` bumped → `_statusPct` re-runs):
  - CustomConfigDoc: `round(1/160×100)` = `round(0.625)` = `1` → **"1%"**
  - others still `"0%"`

## 3.13 `_formatCount` / `_formatStorage` — the value cell

Slot 4 uses `_formatCount(row.value)` (`160 → "160"`, `1500 → "1.5K"`); Slot 5
uses `_formatStorage(row.value)` (`263000 → "256.8 KB"`). These just make the
number compact for display.

---

# PART 4 — THE FULL ORDER OF CALLS (who calls whom)

```
byDocType aggregation arrives
        │  (observer)
        ▼
_computeTopByDocs(byDocType)
        │  builds _docCountByType (map of type -> total)
        ├─► _topN(byDocType, 5)
        │        └─► _friendlyDocType(key)   (per row, makes the label)
        │        returns the shaped top-5 list
        ├─ sets topByDocs, topByDocsMax
        ├─► _rebuildUseCaseTypes()   → sets _useCaseTypes = [{type,mig:0}, ...]
        └─ _statusVersion++
        │
        ▼  (_useCaseTypes changed)
dom-repeat over _useCaseTypes  (uc = this type)
        └─► _typeMigratedQuery(uc.type, startDate, endDate)  → query runs
                 └─ count returns → _onTypeMigratedCount(e)
                        reads e.detail.value, e.model.uc.type
                        updates _typeMigrated, _statusVersion++
        │
        ▼  (topByDocs ready; _statusVersion changing)
dom-repeat over topByDocs  (row = this ranked item)
        ├─► _inc(index)                         → rank number
        ├─  row.label                           → name
        ├─► _barStyle(row.value, topByDocsMax)  → bar width
        ├─► _formatCount(row.value)             → value cell
        └─► _statusPct(row.type, _docCountByType, _statusVersion) → migrated %

(storageByDocType follows the exact same path via _computeTopByStorage → Slot 5,
 reusing _topN, _friendlyDocType, _rebuildUseCaseTypes, and the same row helpers
 with topByStorageMax and _formatStorage.)
```

---

# PART 5 — TWO WORKED EXAMPLES

## Example A — Slot 4 (by document volume)

**Setup.** `byDocType` as in Part 3. No migrated queries have returned yet.

1. Observer fires `_computeTopByDocs(byDocType)`.
2. `_docCountByType = {CustomConfigDoc:160, BackgroundInvestigation:100,
   FFIORMDContractDoc:16, RegulatoryFolder:14, StandardFolder:8}`.
3. `_topN(...,5)` → sorts (already sorted), reshapes, calling `_friendlyDocType`
   for each label. Returns the 5 `{type,label,value}` rows.
4. `topByDocs` = those rows; `topByDocsMax = 160`.
5. `_rebuildUseCaseTypes()` → `_useCaseTypes` = 5 `{type,mig:0}` objects.
6. `_statusVersion++`.
7. Per-type dom-repeat fires 5 migrated queries.
8. The table dom-repeat renders each row:

| index | `_inc` | label | `_barStyle(value,160)` | `_formatCount` | `_statusPct` (before counts) |
|---|---|---|---|---|---|
| 0 | 1 | Custom Config Doc | width:100% | 160 | 0% |
| 1 | 2 | Background Investigation | width:63% | 100 | 0% |
| 2 | 3 | FFIORMD Contract Doc | width:10% | 16 | 0% |
| 3 | 4 | Regulatory Folder | width:9% | 14 | 0% |
| 4 | 5 | Standard Folder | width:5% | 8 | 0% |

9. Then CustomConfigDoc's migrated query returns **1** → `_onTypeMigratedCount`
   sets `_typeMigrated.CustomConfigDoc = 1`, bumps `_statusVersion`. Because the %
   cell binds `_statusVersion`, `_statusPct` re-runs for row 1:
   `round(1/160×100) = 1` → its cell flips from **0% to 1%**. The others (migrated
   0) stay **0%**.

## Example B — Slot 5 (by storage added)

**Setup.**
```
storageByDocType = [
  {key:"CustomConfigDoc", value:263000},
  {key:"Video",           value:120000},
  {key:"Picture",         value:60000},
  {key:"StandardFolder",  value:8000},
  {key:"Note",            value:3000}
]
```

1. Observer fires `_computeTopByStorage(storageByDocType)`.
2. `_topN(...,5)` sorts by storage desc (already sorted), reshapes with
   `_friendlyDocType`. `topByStorage` = 5 rows; `topByStorageMax = 263000`.
3. `_rebuildUseCaseTypes()` merges these types with Slot 4's types (dedup). If new
   types like `Video`, `Picture`, `Note` appeared, `_useCaseTypes` grows and their
   migrated queries fire too.
4. The Slot 5 table renders, using `topByStorageMax` for bars and
   `_formatStorage` for values:

| index | `_inc` | label | `_barStyle(value,263000)` | `_formatStorage` | `_statusPct` |
|---|---|---|---|---|---|
| 0 | 1 | Custom Config Doc | width:100% | 256.8 KB | depends on migrated/total |
| 1 | 2 | Video | width:46% | 117.2 KB | 0% |
| 2 | 3 | Picture | width:23% | 58.6 KB | 0% |
| 3 | 4 | Standard Folder | width:5% (clamped) | 7.8 KB | 0% |
| 4 | 5 | Note | width:4% (clamped) | 2.9 KB | 0% |

Note the **migrated %** here still divides by **document count** from
`_docCountByType` (not by storage), because "migrated %" means share of that
type's *documents* that are migrated — the same number shown in Slot 4 for the
same type.

---

# PART 6 — ONE-PARAGRAPH SUMMARY

When `byDocType` (or `storageByDocType`) arrives, its observer runs
`_computeTopByDocs` (or `_computeTopByStorage`). That builds a `type → total`
lookup (`_docCountByType`, docs only), calls `_topN` to sort and take the top 5 —
which calls `_friendlyDocType` to make each label — stores the list and its max,
then `_rebuildUseCaseTypes` builds `_useCaseTypes` (distinct `{type, mig:0}`
objects). A per-type `dom-repeat` fires one migrated-count query each via
`_typeMigratedQuery`; each result lands in `_onTypeMigratedCount`, which updates
`_typeMigrated` and bumps `_statusVersion`. The visible table `dom-repeat` renders
each `row` with `_inc(index)` for the rank, `row.label` for the name,
`_barStyle(row.value, max)` for the bar, `_formatCount`/`_formatStorage` for the
value, and `_statusPct(row.type, _docCountByType, _statusVersion)` for the
migrated % — which shows 0% immediately and updates to the real percentage as each
type's migrated count arrives.
