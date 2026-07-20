# How to Create Your Own Page Provider (`fid_MIGRATED`) — Step by Step

**What this achieves.** Right now, the "migrated documents" count on your
dashboard is built from a query typed directly into the component's
JavaScript (`_migratedQuery`). That works, but whether it runs fast
(Elasticsearch) or slow (the database) depends on a server setting
(`nuxeo.conf`) that isn't guaranteed and can silently drift. This guide
walks you through creating your **own named page provider in Studio**,
switching it to Elasticsearch with a single checkbox, and testing it
properly **before** touching the dashboard — so if anything goes wrong,
you find out in a safe testing tool (Bruno), not in the live UI.

**Time needed:** roughly 30–45 minutes, done carefully.

**Golden rule for this whole guide:** test in Bruno first. Only move to the
next step once the current one is confirmed working. Never skip ahead.

---

## Part 1 — Create the page provider in Studio

### Step 1.1 — Open your Studio project

1. Go to your Nuxeo Studio project (the same one where you defined
   `fid-analytical-dashboard`, project `fidelity_ctg-csp-new`).
2. Log in if prompted.

### Step 1.2 — Find the Page Providers section

1. Look in Studio's left-hand navigation for **Configuration** (sometimes
   labelled **Configuration** or shown as a gear/settings icon).
2. Inside Configuration, find **Page Providers**.
   - If you already have an old/reverted `fid_MIGRATED` entry from a
     previous attempt, you can reuse it — just double-click to open it and
     skip to Step 1.4. Otherwise, continue to Step 1.3.

### Step 1.3 — Create a new page provider

1. Click **New** (usually a "+" button or a "New" button near the top of
   the Page Providers list).
2. You'll be asked to name it. Enter:
   ```
   fid_MIGRATED
   ```
   (Exact spelling and capitalization matter — this name is what the
   dashboard code will reference later.)
3. Click **Create** / **Save** to confirm the name.

### Step 1.4 — Enter the query

You should now be inside the editor for `fid_MIGRATED`. Look for a tab or
section called **Query & Form** (this is also where the important
checkbox from Step 1.5 lives).

1. Find the field for the **NXQL pattern** (may be labelled "Pattern",
   "Query", or shown as a text box under an "Advanced"/"NXQL" option).
2. Enter this **exactly**, including the two `?` marks:

   ```sql
   SELECT * FROM Document WHERE ecm:isVersion = 0 AND ecm:isTrashed = 0 AND dc:created BETWEEN ? AND ? AND fid_migration:migrationHistory/*/systemName IS NOT NULL
   ```

   **What each part means, in plain terms:**
   - `ecm:isVersion = 0` — ignore old document versions, only count the
     current one.
   - `ecm:isTrashed = 0` — ignore deleted documents.
   - `dc:created BETWEEN ? AND ?` — the date range. The two `?` are
     **placeholders** — empty slots that get filled in later by whoever
     calls this provider (in our case, the dashboard, passing the start
     and end date the user picked).
   - `fid_migration:migrationHistory/*/systemName IS NOT NULL` — the
     actual business rule: "this document has at least one migration
     history entry," i.e. it's been migrated. This part never changes, so
     it has no `?` — it's baked directly into the query.

   **Important — do NOT write `TIMESTAMP ?`.** Just the bare `?`. We are
   deliberately starting with the simpler form, because it's more likely
   to work first try. (If Bruno testing in Part 3 shows this doesn't
   parse, there's a fallback in the Troubleshooting section below.)

3. Save this pattern (there should be a Save button on the tab or panel).

### Step 1.5 — Turn on Elasticsearch — THE key step

This is the single most important click in this whole guide.

1. Still on the **Query & Form** tab, look for a checkbox labelled:
   ```
   Use Elasticsearch index
   ```
2. **Check it.**
3. **Save.**

This tells Nuxeo: "always run this specific query against Elasticsearch,
never against the database." Unlike the server-wide `nuxeo.conf` setting
you've dealt with before, this is saved as part of *this provider's own
definition* — it travels with your Studio project and can't be silently
reverted by someone else's server change.

### Step 1.6 — Deploy your Studio project

1. Find the **Deploy** (or **Publish**) button in Studio — usually near
   the top of the screen, sometimes under a "Deployment" tab.
2. Click it, and confirm if asked.
3. Wait for Studio to confirm the deployment finished (this can take a
   minute or two — Studio is rebuilding and pushing your project to the
   Nuxeo server).

**Do not skip this step.** Changes made in Studio only exist in the
browser until you deploy — the server doesn't know about `fid_MIGRATED`
until deployment finishes.

---

## Part 2 — Confirm the deployment landed

Before testing the query itself, make sure the provider actually exists on
the server.

1. Open your Nuxeo server in the browser:
   ```
   http://localhost:8185/nuxeo/
   ```
2. Log in as Administrator.
3. Go to **Admin Center → Elasticsearch → Page Providers** (or similar —
   look for a tab called "Page Provider" under the Elasticsearch section).
4. Look for `fid_MIGRATED` in the list.
   - **Found it?** Good — proceed to Part 3.
   - **Not there?** The deployment likely didn't finish, or didn't reach
     this server. Go back to Studio, confirm the deploy really completed
     (check for a success message / green checkmark), and try again.

---

## Part 3 — Test in Bruno (do this BEFORE touching the dashboard)

This is the safety step. We prove the provider works correctly in a
throwaway testing tool first, so if something's wrong, you find out here —
not by breaking the live dashboard.

### Step 3.1 — Pick a known date range and a known answer

Choose a date range you already know the migrated count for (from the
current, working dashboard). For example: **July 2026**, and you know from
the dashboard that this currently shows **5 migrated documents** (adjust to
your real numbers).

Write this down — you'll compare against it in Step 3.3.

### Step 3.2 — Build the Bruno request

Create a new request in Bruno:

- **Method:** `GET`
- **URL:**
  ```
  http://localhost:8185/nuxeo/api/v1/search/pp/fid_MIGRATED/execute
  ```
- **Query params** (add these in Bruno's "Params" tab):

  | Key | Value |
  |---|---|
  | `queryParams` | `2026-07-01 00:00:00,2026-07-31 23:59:59` |
  | `pageSize` | `1` |

  **What `queryParams` is doing:** it's a comma-separated list, in order,
  of the values that fill in the two `?` placeholders from Step 1.4 — the
  first value fills the first `?` (start date), the second fills the
  second `?` (end date).

  **What `pageSize=1` is doing:** exactly what we discussed earlier in
  this project — fetch only 1 document, but still get the accurate total
  count back. This keeps the test request itself cheap.

- **Auth:** use the same Basic Auth (Administrator credentials) you've
  used for every other Bruno request in this project.

### Step 3.3 — Send it and read the response

Click **Send**.

**Look at the `resultsCount` field in the response JSON.**

- **It matches your known number (e.g. `5`)?** 🎉 Success — the query
  works, the placeholders are filled correctly, and it's running on
  Elasticsearch. Skip to Part 4.
- **It's `0`, or a different number than expected?** Something's off with
  the query or the date format — see Troubleshooting below.
- **You get an error (400/500) instead of a JSON count?** The query
  didn't parse — see Troubleshooting below.

### Step 3.4 — Troubleshooting the Bruno test

**If you get a parse error, or the count is wrong:**

Try changing the Studio query pattern (back in Part 1, Step 1.4) to
explicitly mark the placeholders as timestamps:

```sql
SELECT * FROM Document WHERE ecm:isVersion = 0 AND ecm:isTrashed = 0 AND dc:created BETWEEN TIMESTAMP ? AND TIMESTAMP ? AND fid_migration:migrationHistory/*/systemName IS NOT NULL
```

Save, re-deploy (Step 1.6), and repeat the Bruno test (Step 3.2–3.3). One
of these two forms (plain `?` or `TIMESTAMP ?`) should work — this is the
one part of the setup that genuinely can't be predicted in advance and has
to be tried both ways.

**If you get "could not resolve page provider" / 404:**
The deployment hasn't landed yet, or the name is misspelled somewhere.
Recheck Part 2, and double-check `fid_MIGRATED` is spelled identically
(case-sensitive) in the URL and in Studio.

**If `resultsCount` is present but the count seems too low/high:**
Double-check the date range you're comparing against — make sure you're
comparing like-for-like (same start/end dates as what the dashboard is
currently showing for its "known" number).

**Do not proceed to Part 4 until `resultsCount` matches your known
number.** This is the whole point of testing in Bruno first.

---

## Part 4 — Wire it into the dashboard (only after Part 3 succeeds)

Once Bruno confirms the count is correct, make these two changes in
`fid-analytical-dashboard-query.html`.

### Step 4.1 — Replace the migrated data element

**Find this block** (search for `Migrated documents in the period`):

```html
<!-- Migrated documents in the period: migrationHistory has at least one entry -->
<nuxeo-page-provider
  auto
  page-size="1"
  query="[[_migratedQuery(startDate, endDate)]]"
  current-page="{{_ignoredMigratedPage}}"
  results-count="{{migratedCount}}">
</nuxeo-page-provider>
```

**Replace it with:**

```html
<!-- Migrated documents in the period: uses the Studio page provider
     fid_MIGRATED, which is explicitly configured to use Elasticsearch
     (checked in Studio's Query & Form tab). The date range is passed as
     positional params matching the two ? placeholders in the query. -->
<nuxeo-page-provider
  auto
  page-size="1"
  provider="fid_MIGRATED"
  params="[[_migratedParams(startDate, endDate)]]"
  current-page="{{_ignoredMigratedPage}}"
  results-count="{{migratedCount}}">
</nuxeo-page-provider>
```

**What changed:** `query="[[_migratedQuery(...)]]"` (the query text built
in JavaScript) is replaced by `provider="fid_MIGRATED"` (the name of your
new Studio provider) plus `params="..."` (the values to fill its `?`
placeholders). `results-count="{{migratedCount}}"` — the part that
actually feeds your dashboard's numbers — is untouched, so nothing
downstream (the donut chart, the migration table, the email) needs to
change at all.

### Step 4.2 — Add the `_migratedParams` helper

**Find this method** (search for `_migratedQuery: function`):

```javascript
_migratedQuery: function(startDate, endDate) {
  var range = this._dateRangeClause(startDate, endDate);
  if (!range) return '';
  return "SELECT * FROM Document WHERE ecm:isVersion = 0 " +
         "AND ecm:isTrashed = 0 " + range +
         " AND fid_migration:migrationHistory/*/systemName IS NOT NULL";
},
```

**Leave it exactly as it is** (don't delete it — see the rollback note at
the end of this guide). **Add this new method directly after it:**

```javascript
// Positional parameters for the fid_MIGRATED Studio page provider.
// ORDER matches the two ? placeholders in the Studio query:
//   1st ? = start timestamp, 2nd ? = end timestamp.
// Uses the same 'YYYY-MM-DD HH:MM:SS' format proven to work in Bruno.
_migratedParams: function(startDate, endDate) {
  if (!startDate || !endDate) { return JSON.stringify([]); }
  var start = this._toTimestamp(startDate, false);   // 'YYYY-MM-DD 00:00:00'
  var end   = this._toTimestamp(endDate, true);       // 'YYYY-MM-DD 23:59:59'
  return JSON.stringify([start, end]);
},
```

This reuses `_toTimestamp`, a helper already in the file — it formats a
date as `'YYYY-MM-DD 00:00:00'` for a start date or `'YYYY-MM-DD
23:59:59'` for an end date, exactly matching what you typed into Bruno's
`queryParams` field in Step 3.2.

**If your Bruno test in Part 3 needed the `TIMESTAMP ?` form instead of
plain `?`:** no change needed here — `_migratedParams` only supplies the
*values*; whether the Studio query wraps them in `TIMESTAMP ...` is
decided entirely by the Studio query pattern from Part 1.

### Step 4.3 — Save and deploy the dashboard file

Deploy `fid-analytical-dashboard-query.html` the way you normally do, then
**hard-refresh** the browser (Ctrl+Shift+R, or open an incognito window) to
make sure you're not looking at a cached copy.

---

## Part 5 — Final verification

1. Open the dashboard, select the **same date range** you tested in
   Bruno (e.g. July 2026).
2. Check the migrated count / donut chart / migration table.
3. **It should show the exact same number Bruno gave you.** If it does,
   you're done — the dashboard's migrated count is now guaranteed to run
   on Elasticsearch, independent of any server-wide setting.

**Bonus check (optional):** open DevTools → Network, reload, and confirm
the request for `fid_MIGRATED` returns quickly and its `resultsCount`
matches what's on screen.

---

## Rollback — if anything goes wrong

Because Step 4.2 told you to **keep** `_migratedQuery` rather than delete
it, reverting is a two-line change:

1. In the template, change `provider="fid_MIGRATED" params="..."` back to
   `query="[[_migratedQuery(startDate, endDate)]]"`.
2. Remove (or just ignore) the new `_migratedParams` method.

That's it — you're back to exactly the working state you had before
starting this guide. Nothing else on the dashboard depends on this
change, so rollback is safe and isolated.

---

## Quick checklist

- [ ] Studio: created `fid_MIGRATED` page provider
- [ ] Studio: entered the NXQL pattern with two `?` placeholders
- [ ] Studio: checked **"Use Elasticsearch index"**
- [ ] Studio: saved and **deployed** the project
- [ ] Admin Center: confirmed `fid_MIGRATED` appears under Elasticsearch → Page Providers
- [ ] Bruno: tested `fid_MIGRATED/execute` with a known date range
- [ ] Bruno: `resultsCount` matched the known correct number
- [ ] Dashboard: replaced the `<nuxeo-page-provider>` element (Step 4.1)
- [ ] Dashboard: added `_migratedParams` (Step 4.2)
- [ ] Dashboard: deployed, hard-refreshed, confirmed the on-screen count matches Bruno's number

If every box is checked, you're done.
