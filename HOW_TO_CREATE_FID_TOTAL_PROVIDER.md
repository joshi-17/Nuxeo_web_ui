# How to Create the `fid_TOTAL` Page Provider — Total Document Count

**What this achieves.** You already have `fid_MIGRATED` working in Bruno — a
Studio page provider, running on Elasticsearch, that counts *migrated*
documents in a date range. This guide creates its twin, **`fid_TOTAL`**,
which counts **all** documents in a date range (no migration filter). Once
both exist, your dashboard's two core numbers — total and migrated — will
both come from named, Elasticsearch-backed providers, and the migrated %
(migrated ÷ total) will be built entirely on ES-backed counts.

**Why a second provider and not reuse `fid_MIGRATED`:** `fid_MIGRATED` has
the migration filter baked into its fixed part — every call it makes carries
`fid_migration:migrationHistory/*/systemName IS NOT NULL`, and there's no way
to switch that off per-call. The total count needs the *same* query *without*
that one line, so it needs its own provider. It is otherwise identical.

**Good news:** because it's `fid_MIGRATED` minus one line, this is the easy
one — you already solved every hard part (the SELECT-prefix issue, the
parameter style, the predicate syntax) getting `fid_MIGRATED` working. Copy
that success.

**Time needed:** ~15 minutes, since you've done this once already.

---

## Part 1 — Create `fid_TOTAL` in Studio

### Step 1.1 — Duplicate or create

The fastest route: if Studio lets you **duplicate** `fid_MIGRATED`, do that
and rename the copy to `fid_TOTAL` — then you only have to remove one
predicate (Step 1.3). If duplication isn't available, create a new page
provider from scratch named exactly:
```
fid_TOTAL
```
(Exact spelling and capitalization — the dashboard code will reference this.)

### Step 1.2 — Set up the query, EXACTLY like `fid_MIGRATED`

Rebuild the same structure that worked for `fid_MIGRATED`:

- The **same fixed part**, but **without** the migration predicate.
- The **same two date predicates** on `dc:created` (the greater-than-or-equal
  and less-than-or-equal pair) that you already got working.
- The **same parameter style** (named parameters — the approach that finally
  worked for `fid_MIGRATED`, not raw `?` placeholders).

The fixed part for `fid_TOTAL` is just:
```
ecm:isVersion = 0 AND ecm:isTrashed = 0
```

That's the **only** difference from `fid_MIGRATED`. In `fid_MIGRATED` the
fixed part was:
```
ecm:isVersion = 0 AND ecm:isTrashed = 0 AND fid_migration:migrationHistory/*/systemName IS NOT NULL
```
For `fid_TOTAL`, delete the ` AND fid_migration:migrationHistory/*/systemName IS NOT NULL`
part. Everything else — the two `dc:created` date predicates, the parameter
names, the operators — stays identical.

### Step 1.3 — If you duplicated

If you copied `fid_MIGRATED`, just open the copy and **remove the migration
predicate** from the fixed clause, leaving `ecm:isVersion = 0 AND
ecm:isTrashed = 0` plus the two date predicates. Save.

### Step 1.4 — Turn on Elasticsearch

Exactly as you did for `fid_MIGRATED`: on the **Query & Form** tab, check:
```
Use Elasticsearch index
```
Save. (If you duplicated `fid_MIGRATED`, this may already be checked — but
**verify it is**, don't assume.)

### Step 1.5 — Deploy

Deploy your Studio project, same as before. Wait for the deploy to finish.

---

## Part 2 — Test `fid_TOTAL` in Bruno (before touching the dashboard)

Same discipline that worked for `fid_MIGRATED`. Prove it in Bruno first.

### Step 2.1 — Know your expected number

From the current working dashboard, note the **total** documents for a known
period — say July 2026 shows **21,000** total (use your real number).

### Step 2.2 — The Bruno request

Mirror your working `fid_MIGRATED` request exactly, changing only the
provider name in the URL:

- **Method:** `GET`
- **URL:**
  ```
  http://localhost:8185/nuxeo/api/v1/search/pp/fid_TOTAL/execute
  ```
- **Query params** — use the **same parameter names** that worked for
  `fid_MIGRATED` (whatever Studio generated for your two date predicates —
  e.g. `dc_created_min` / `dc_created_max`), plus `pageSize`:

  | Key | Value |
  |---|---|
  | (your start-date param name) | `2026-07-01 00:00:00` |
  | (your end-date param name) | `2026-07-31 23:59:59` |
  | `pageSize` | `1` |

- **Auth:** same Basic Auth as always.

**Tip:** literally copy your working `fid_MIGRATED` Bruno request, change
`fid_MIGRATED` → `fid_TOTAL` in the URL, and send. The params are identical.

### Step 2.3 — Check the result

Look at `resultsCount`:
- **Matches your known total (e.g. 21,000)?** Done — `fid_TOTAL` works.
- **Returns the *migrated* count instead?** You forgot to remove the
  migration predicate — go back to Step 1.3.
- **Error / wrong param names?** Recheck the parameter names match what
  Studio generated (compare against your working `fid_MIGRATED` request).

**A useful sanity check:** `fid_TOTAL`'s count should be **larger** than
`fid_MIGRATED`'s count for the same period (total ≥ migrated, always). If
`fid_TOTAL` returns the same or a smaller number, the migration predicate is
probably still in there.

---

## Part 3 — Wire both providers into the dashboard

Now switch both count queries to the named providers. Two elements to
replace, two helper methods to add.

### Step 3.1 — Replace the TOTAL element

**Find** (search for `Total documents in the period`):
```html
<nuxeo-page-provider
  auto
  page-size="1"
  query="[[_totalQuery(startDate, endDate)]]"
  current-page="{{_ignoredTotalPage}}"
  results-count="{{totalCount}}">
</nuxeo-page-provider>
```

**Replace with:**
```html
<!-- Total documents in the period — Studio page provider fid_TOTAL,
     configured to use Elasticsearch. Dates passed as named params. -->
<nuxeo-page-provider
  auto
  page-size="1"
  provider="fid_TOTAL"
  params="[[_totalParams(startDate, endDate)]]"
  current-page="{{_ignoredTotalPage}}"
  results-count="{{totalCount}}">
</nuxeo-page-provider>
```

### Step 3.2 — Replace the MIGRATED element

**Find** (search for `Migrated documents in the period`):
```html
<nuxeo-page-provider
  auto
  page-size="1"
  query="[[_migratedQuery(startDate, endDate)]]"
  current-page="{{_ignoredMigratedPage}}"
  results-count="{{migratedCount}}">
</nuxeo-page-provider>
```

**Replace with:**
```html
<!-- Migrated documents in the period — Studio page provider fid_MIGRATED,
     configured to use Elasticsearch. Dates passed as named params. -->
<nuxeo-page-provider
  auto
  page-size="1"
  provider="fid_MIGRATED"
  params="[[_migratedParams(startDate, endDate)]]"
  current-page="{{_ignoredMigratedPage}}"
  results-count="{{migratedCount}}">
</nuxeo-page-provider>
```

### Step 3.3 — Add the two params helpers

**IMPORTANT — the parameter format depends on how your Studio provider
expects them.** Studio named-parameter providers are called over REST with
`fieldname=value` query-string parameters. In the Polymer
`<nuxeo-page-provider>` element, the `params` attribute maps to those. You
have two possible shapes depending on your Studio setup — **use whichever
matched your working Bruno request**:

**If your Bruno test used named params** (e.g. `?dc_created_min=...&dc_created_max=...`),
the element needs **named parameters**, supplied as a JSON object via the
`params` attribute is not correct — instead the element uses a
`named-parameters` style. Add these helpers, which build the value strings,
and see the note below on wiring:

```javascript
// Timestamp strings for the fid_TOTAL / fid_MIGRATED date predicates.
// Reuses _toTimestamp (already in the file): start -> 'YYYY-MM-DD 00:00:00',
// end -> 'YYYY-MM-DD 23:59:59' — the exact format proven in Bruno.
_totalParams: function(startDate, endDate) {
  if (!startDate || !endDate) { return JSON.stringify([]); }
  return JSON.stringify([
    this._toTimestamp(startDate, false),
    this._toTimestamp(endDate, true)
  ]);
},

_migratedParams: function(startDate, endDate) {
  if (!startDate || !endDate) { return JSON.stringify([]); }
  return JSON.stringify([
    this._toTimestamp(startDate, false),
    this._toTimestamp(endDate, true)
  ]);
},
```

**On matching the wiring to your provider type — read this carefully:**
- The `params="[[...]]"` attribute (a JSON array) is for **positional**
  page-provider parameters.
- If Studio built your providers with **named parameters** (the fieldname
  style you tested in Bruno), the Polymer element instead needs the
  attribute `named-params` or the query-parameters passed differently.

Because your Bruno test is the ground truth, do this: **tell me the exact
Bruno URL that worked** (the full query string with the parameter names),
and I'll give you the precise `params` / `named-params` wiring to match it
one-to-one. The helpers above supply the two timestamp values; the only
open question is the attribute name the element uses to hand them to a
named-parameter provider, and that's determined by your working Bruno call.

### Step 3.4 — Keep the old helpers (for rollback)

**Do NOT delete** `_totalQuery` or `_migratedQuery`. Leave them in the file.
If anything misbehaves, reverting is just changing the two elements back to
`query="[[_totalQuery(...)]]"` / `query="[[_migratedQuery(...)]]"`. Costs
nothing to keep them; saves you if you need to roll back fast.

---

## Part 4 — Deploy and verify

1. Deploy `fid-analytical-dashboard-query.html`, hard-refresh (Ctrl+Shift+R).
2. Select the same period you tested in Bruno.
3. Confirm:
   - **Total count** matches `fid_TOTAL`'s Bruno number.
   - **Migrated count** matches `fid_MIGRATED`'s Bruno number.
   - **Migrated %** (the donut) = migrated ÷ total, computed from the two.
4. Optional: DevTools → Network, reload, confirm both `fid_TOTAL` and
   `fid_MIGRATED` requests fire and return quickly.

---

## Rollback

Both old query-builder methods are still in the file (Step 3.4), so to
revert either count:
1. Change `provider="fid_TOTAL" params="..."` back to
   `query="[[_totalQuery(startDate, endDate)]]"` (and/or the migrated one).
2. Ignore the unused `_totalParams` / `_migratedParams` methods.

Safe and isolated — nothing else depends on these two changes.

---

## Checklist

- [ ] Studio: created `fid_TOTAL` (duplicate of `fid_MIGRATED` minus the migration predicate)
- [ ] Studio: same two `dc:created` date predicates, same parameter names
- [ ] Studio: checked **"Use Elasticsearch index"**
- [ ] Studio: saved and deployed
- [ ] Bruno: tested `fid_TOTAL/execute` — `resultsCount` matches known total
- [ ] Bruno: sanity check — `fid_TOTAL` count ≥ `fid_MIGRATED` count for same period
- [ ] Dashboard: replaced both `<nuxeo-page-provider>` elements
- [ ] Dashboard: added `_totalParams` + `_migratedParams`
- [ ] Dashboard: sent me the working Bruno URL so the params wiring can be confirmed
- [ ] Dashboard: deployed, hard-refreshed, both counts + migrated % correct

---

## Note on the other queries in the dashboard

Your goal is "all named custom providers." Two more categories exist in the
file, worth being clear about:

1. **The `<nuxeo-repository-data>` chart feeds** (growth, storage, baseline)
   — these are aggregation widgets, always Elasticsearch by nature, and are
   a *different* mechanism from page providers. They don't need a named
   provider and can't use one in the same way; they're already ES-backed.
   Leave them as they are.

2. **The per-month and per-type migrated counts** (the small repeated
   queries) — these currently use ad-hoc `query=`. If you want *these* on
   `fid_MIGRATED` too, they can pass different date params to the same
   provider — but that's a follow-up once total + migrated are confirmed
   working. Get these two solid first, then we extend the pattern.

Once `fid_TOTAL` and `fid_MIGRATED` are both live and verified, send me the
working Bruno URL and we'll lock in the exact params wiring and move on to
the per-month counts if you want full named-provider coverage.
