# How to Create `fid_MIGRATED_BY_TYPE` — Per-Type Migrated Count Provider

**What this achieves.** Slots 4 & 5 (Top 5 use cases) show a migration % per
document type. Right now those per-type migrated counts run as ad-hoc NXQL,
which is only Elasticsearch-backed if the `nuxeo.conf` override line is set.
This guide creates a third Studio provider — `fid_MIGRATED` plus ONE extra
predicate on the document type — so the per-type counts are **guaranteed
Elasticsearch** via the Studio checkbox, exactly like `fid_MIGRATED` and
`fid_TOTAL`. The dashboard then calls it with the same proven `fetch()`
pattern that just fixed the monthly chart.

**Time needed:** ~15 minutes. You've done this twice already — this is
`fid_MIGRATED` + one predicate.

---

## Part 1 — Create the provider in Studio

### Step 1.1 — Duplicate `fid_MIGRATED`

If Studio lets you duplicate, duplicate `fid_MIGRATED` and rename the copy:
```
fid_MIGRATED_BY_TYPE
```
(Exact spelling/capitalization — the dashboard code references this name.)

If duplication isn't available, create it new and rebuild the same structure
as `fid_MIGRATED`:
- Fixed part: `ecm:isVersion = 0 AND ecm:isTrashed = 0 AND fid_migration:migrationHistory/*/systemName IS NOT NULL`
- The same two `dc:created` date predicates (>= and <=) that generated the
  `dublincore_created` / `dublincore_created_1` parameters.

### Step 1.2 — Add ONE new predicate: the document type

Using the same guided predicate editor you used for the dates:

1. Add a new predicate.
2. **Field:** `ecm:primaryType`
3. **Operator:** equals (`=`)
4. Save.

Studio will generate a parameter name for it — following the same naming
pattern as your date params, it will most likely be:
```
ecm_primaryType
```
**IMPORTANT: note down the EXACT generated parameter name.** If it's anything
other than `ecm_primaryType`, you'll change one line in the dashboard code
(clearly marked — see Part 3).

### Step 1.3 — Verify "Use Elasticsearch index" is checked

On the Query & Form tab — if you duplicated, it may already be checked, but
**verify, don't assume**. This checkbox is the entire point: it guarantees ES.

### Step 1.4 — Deploy the Studio project

Deploy and wait for it to finish, same as before.

---

## Part 2 — Test in Bruno (before touching the dashboard)

### Step 2.1 — Know your expected number

Pick a type you can see in the dashboard's Top 5 (e.g. `File`) and a period,
and note the migration % / count currently shown (from the working ad-hoc
version) so you can compare.

### Step 2.2 — The Bruno request

Copy your working `fid_MIGRATED` request, change the provider name, and add
the type param:

```
GET http://localhost:8185/nuxeo/api/v1/search/pp/fid_MIGRATED_BY_TYPE/execute
```

| Key | Value |
|---|---|
| `dublincore_created` | `2026-07-01 00:00:00` |
| `dublincore_created_1` | `2026-07-31 23:59:59` |
| `ecm_primaryType` (or whatever Studio generated) | `File` |
| `pageSize` | `1` |

Same Basic Auth as always.

### Step 2.3 — Check the result

- **`resultsCount` matches the expected per-type migrated count?** Done.
- **Sanity check:** the count must be **≤** what `fid_MIGRATED` returns for
  the same period (a single type's migrated docs can't exceed all migrated
  docs). If it's equal for every type, the type predicate isn't filtering —
  check the parameter name matches exactly.
- **Error / same count for all types?** The param name in your request
  doesn't match what Studio generated — check Step 1.2's noted name.

**Do not proceed to Part 3 until Bruno confirms correct counts for at least
two different types.**

---

## Part 3 — The dashboard code (already changed for you)

The updated `fid-analytical-dashboard-FALLBACK.html` already contains the
fetch-based per-type implementation. Only ONE thing may need adjusting:

In the method `_fetchTypeMigrated`, there is a clearly marked line:

```javascript
var TYPE_PARAM = 'ecm_primaryType';   // <-- CHANGE HERE if Studio generated a different name
```

If your Bruno test needed a different parameter name, change that single
string to match, redeploy the dashboard file, hard-refresh. That's the only
code adjustment this provider can require.

---

## Part 4 — Deploy and verify

1. Deploy the updated dashboard file, hard-refresh (Ctrl+Shift+R).
2. Check slots 4 & 5: each type's migration % should match what you saw
   before (the ad-hoc version was correct — this change is about guaranteeing
   ES, not changing numbers).
3. DevTools → Network → filter `fid_MIGRATED_BY_TYPE`: you should see one
   request per distinct type (≤10), each with the three params visible in the
   URL and a sensible `resultsCount` in the response.

---

## Checklist

- [ ] Studio: `fid_MIGRATED_BY_TYPE` created (fid_MIGRATED + `ecm:primaryType =` predicate)
- [ ] Studio: noted the exact generated type-parameter name
- [ ] Studio: "Use Elasticsearch index" checked
- [ ] Studio: deployed
- [ ] Bruno: correct `resultsCount` for at least two different types
- [ ] Bruno: per-type count ≤ fid_MIGRATED count for the same period
- [ ] Code: `TYPE_PARAM` matches the Studio-generated name (change if needed)
- [ ] Dashboard: deployed, hard-refreshed, slots 4 & 5 percentages correct
- [ ] Network tab: one `fid_MIGRATED_BY_TYPE` request per type, params visible

Once every box is checked, **every migration-history query in the dashboard
is guaranteed Elasticsearch** — total, migrated, per-month, and per-type all
via Studio providers with the ES checkbox, and all charts via ES aggregations.
The `nuxeo.conf` override line is no longer needed by this dashboard at all.
