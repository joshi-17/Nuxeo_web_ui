# Setting `migrationHistory` via the REST API (Bruno)

A step-by-step guide to mark a Nuxeo document as **migrated** by populating its
`fid_migration:migrationHistory` field through the REST API, using Bruno. This
lets you see the dashboard's migrated count, green donut slice, and green monthly
bar appear.

---

## Why this is needed

A document counts as **migrated** in the dashboard only if it satisfies:

```
fid_migration:migrationHistory/*1/systemName IS NOT NULL
```

i.e. it must have at least one migration-history entry with a `systemName` set.

A document created via the Nuxeo Web UI **+** button (or drag-and-drop) has an
**empty** `migrationHistory`, so it is **non-migrated**. To see a migrated
document, something must explicitly populate `migrationHistory`. This guide does
that manually via REST so you can watch the dashboard react.

The field (from the schema): `fid_migration:migrationHistory[]` — a multi-valued
**Complex** type with sub-fields `systemName`, `systemDocId`, `ingestionDate`,
`comment`. Migrated = array non-empty; non-migrated = array empty.

---

## Bruno setup

### 1. Create an Environment

Add these variables:

| Variable    | Value                          |
|-------------|--------------------------------|
| `baseUrl`   | `http://localhost:8185/nuxeo`  |
| `username`  | `Administrator`                |
| `password`  | `Administrator` (or your admin password) |

### 2. Set collection-level auth

Set **Auth → Basic** at the collection level, using `{{username}}` and
`{{password}}`, so every request inherits it.

---

## Request 1 — Find a document to tag

Get the UID of a document you'll mark as migrated.

**Method / URL**
```
GET  {{baseUrl}}/api/v1/search/lang/NXQL/execute?query=SELECT * FROM Document WHERE ecm:primaryType = 'File' AND ecm:isVersion = 0 AND ecm:isTrashed = 0&pageSize=5
```

**Headers**
```
Accept: application/json
```

**What to do with the response:** find an entry under `entries`, copy its `uid`
value. Call it `<DOC_UID>` — you'll use it in the next requests.

> Tip: pick a document whose creation month you can recognise, so you know which
> bar on the Monthly Migration chart should turn green.

---

## Request 2 — Set `migrationHistory` on that document

This writes one migration-history entry, which makes the document migrated.

**Method / URL**
```
PUT  {{baseUrl}}/api/v1/id/<DOC_UID>
```

**Headers**
```
Content-Type: application/json
Accept: application/json
```

**Body (raw JSON)**
```json
{
  "entity-type": "document",
  "uid": "<DOC_UID>",
  "properties": {
    "fid_migration:migrationHistory": [
      {
        "systemName": "LegacySystemA",
        "systemDocId": "DOC-12345",
        "ingestionDate": "2026-06-15T00:00:00.000Z",
        "comment": "Test migration entry via REST"
      }
    ]
  }
}
```

This replaces the empty `migrationHistory` array with one entry that has a
`systemName` — exactly what the dashboard query checks for.

> Note: this PUT replaces the whole `migrationHistory` array with what you send.
> To add multiple entries, include multiple objects in the array.

---

## Request 3 — Verify it saved

Read the document back and confirm the field is populated.

**Method / URL**
```
GET  {{baseUrl}}/api/v1/id/<DOC_UID>
```

**Headers** (the `X-NXproperties` header tells Nuxeo to return all schema
properties, not just the defaults)
```
X-NXproperties: *
Accept: application/json
```

**What to check:** in the response, under `properties`, find
`fid_migration:migrationHistory`. You should see your array with the `systemName`
set.

---

## See it on the dashboard

Reload the dashboard and hard-refresh (Ctrl+Shift+R). The document you tagged
should now count as **migrated**:

- the **Migration Adoption donut** gets a green slice and the centre % rises above
  0,
- the **From migrated use cases** KPI ticks up,
- the **Monthly Migration** chart shows a green (Migrated) bar in the month that
  document was created.

To make the effect more visible, repeat Request 2 on a few more documents (ideally
across different months) so several green bars appear.

---

## Troubleshooting

**The schema prefix might differ.** This guide uses
`fid_migration:migrationHistory`. If your actual prefix/name is different, the PUT
will silently ignore the unknown property. Confirm the exact property key from
Request 3's output (look at the keys under `properties`) and use that everywhere.

**Date format.** Nuxeo dates expect ISO-8601 with a `Z`, e.g.
`2026-06-15T00:00:00.000Z`. If the PUT rejects `ingestionDate`, check the format
first.

**401 Unauthorized.** Re-check the Basic auth credentials and that the
collection-level auth is applied to the request.

**Property not returned in Request 3.** Make sure the `X-NXproperties: *` header
is present; without it, Nuxeo returns only a subset of properties and you may not
see `migrationHistory`.

**Changes not visible on the dashboard.** Hard-refresh the browser
(Ctrl+Shift+R). The dashboard's counts are period-scoped, so also make sure the
document's creation date falls inside the selected period.

---

## Quick reference

| Step | Method | URL | Purpose |
|------|--------|-----|---------|
| 1 | GET | `{{baseUrl}}/api/v1/search/lang/NXQL/execute?query=...` | find a document UID |
| 2 | PUT | `{{baseUrl}}/api/v1/id/<DOC_UID>` | set `migrationHistory` |
| 3 | GET | `{{baseUrl}}/api/v1/id/<DOC_UID>` (with `X-NXproperties: *`) | verify it saved |
