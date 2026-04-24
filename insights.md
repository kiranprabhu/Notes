# Jira Insights API — Manual Test Calls

Use these curl commands to verify the Insights REST API before implementing the sync.
Run them in order — each step uses the ID found in the previous response.

## Variables

Replace these in every command below:

| Placeholder      | Example                        |
|------------------|-------------------------------|
| `JIRA_URL`       | `http://jira.example.com`     |
| `USERNAME`       | `svc-brew`                    |
| `PASSWORD`       | `yourpassword`                |
| `SCHEMA_ID`      | found in Step 1               |

---

## Step 1 — List Insight Schemas

Find the ID for `Dome & CNS Components`.

```bash
curl -s -u "USERNAME:PASSWORD" \
  "http://JIRA_URL/rest/insight/1.0/objectschema/list" \
  | python3 -m json.tool
```

Look for:
```json
{
  "id": 3,
  "name": "Dome & CNS Components",
  ...
}
```

Note the `"id"` — this is your `SCHEMA_ID`.

---

## Step 2 — List Object Types in Schema

Verify `DOME Configuration` and `CNS Configuration` exist.

```bash
curl -s -u "USERNAME:PASSWORD" \
  "http://JIRA_URL/rest/insight/1.0/objecttype/list?objectSchemaId=SCHEMA_ID" \
  | python3 -m json.tool
```

Look for entries with `"name": "DOME Configuration"` and `"name": "CNS Configuration"`.
Note their `"id"` values if you need them for further filtering.

---

## Step 3 — Fetch Sample Config Objects (DOME)

Fetch 3 objects with all attributes expanded. This is the exact call the sync uses.

```bash
curl -s -u "USERNAME:PASSWORD" \
  "http://JIRA_URL/rest/insight/1.0/iql/objects?iql=objectType%3D%22DOME+Configuration%22&objectSchemaId=SCHEMA_ID&resultPerPage=3&startIndex=0&includeAttributes=true&includeAttributesDeep=1" \
  | python3 -m json.tool
```

Decoded IQL: `objectType="DOME Configuration"`

### What to check in the response

**Pagination fields** (top level):
```json
{
  "objectEntries": [...],
  "totalFilterCount": 312,
  "startIndex": 0,
  "toIndex": 2
}
```

**One object entry** (inside `objectEntries[]`):
```json
{
  "id": "1234",
  "label": "FIX Session Config",
  "objectType": { "id": 7, "name": "DOME Configuration" },
  "attributes": [
    {
      "objectTypeAttribute": { "name": "No" },
      "objectAttributeValues": [
        { "displayValue": "42" }
      ]
    },
    {
      "objectTypeAttribute": { "name": "Name" },
      "objectAttributeValues": [
        { "displayValue": "FIX Session Config" }
      ]
    },
    {
      "objectTypeAttribute": { "name": "Description" },
      "objectAttributeValues": [
        { "displayValue": "FIX connectivity settings for OrderRouter" }
      ]
    },
    {
      "objectTypeAttribute": { "name": "Application" },
      "objectAttributeValues": [
        { "displayValue": "OrderRouter" },
        { "displayValue": "RiskEngine" }
      ]
    }
  ]
}
```

### Key things to confirm

1. **Is `Application` a plain text value or an object reference?**
   - Plain text → `objectAttributeValues[].displayValue` is populated
   - Object reference → `objectAttributeValues[].referencedObject.label` is populated, `displayValue` may be null

2. **Is `totalFilterCount` present?** — used for pagination stop condition.

3. **Are there any extra attributes** beyond `No`, `Name`, `Description`, `Application`?

---

## Step 4 — Fetch Sample Config Objects (CNS)

Same call for CNS objects:

```bash
curl -s -u "USERNAME:PASSWORD" \
  "http://JIRA_URL/rest/insight/1.0/iql/objects?iql=objectType%3D%22CNS+Configuration%22&objectSchemaId=SCHEMA_ID&resultPerPage=3&startIndex=0&includeAttributes=true&includeAttributesDeep=1" \
  | python3 -m json.tool
```

---

## Step 5 — Fetch Both Types in One Call (production query)

This is the exact IQL the sync will run:

```bash
curl -s -u "USERNAME:PASSWORD" \
  "http://JIRA_URL/rest/insight/1.0/iql/objects?iql=objectType+in+%28%22DOME+Configuration%22%2C+%22CNS+Configuration%22%29+ORDER+BY+Name+ASC&objectSchemaId=SCHEMA_ID&resultPerPage=5&startIndex=0&includeAttributes=true&includeAttributesDeep=1" \
  | python3 -m json.tool
```

Decoded IQL: `objectType in ("DOME Configuration", "CNS Configuration") ORDER BY Name ASC`
