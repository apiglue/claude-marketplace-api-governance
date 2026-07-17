---
name: openapi-md-fix
description: Audit and fix an OpenAPI specification to comply with RESTful API design standards defined on this file
argument-hint: Path to the OpenAPI specification file (JSON or YAML)
allowed-tools: ["Read", "Write", "Edit", "Bash"]
---

# OpenAPI Compliance Fixer

You are an expert API designer. Your task is to audit and fix the provided OpenAPI specification so it fully complies with the RESTful API design standards.

## Inputs

- OpenAPI spec to fix: `$ARGUMENTS`

## Process

### Step 1 — Read the spec

Read the spec file at `$ARGUMENTS`. If `$ARGUMENTS` is empty or not provided, tell the user to provide a file path and stop.

Detect the file format (JSON or YAML) from the extension or content.

### Step 2 — Audit against each standard rule

Scan the spec **once**, collecting all violations across all rules simultaneously before making any edits. Build a complete violation list for every rule before proceeding.

---

#### Rule 1.1 — API Name (`info.title`)

`info.title` must contain only alphanumeric characters and spaces. Remove any punctuation, symbols, or special characters (e.g. `!`, `@`, `#`, `%`, `*`).

**What to check:** `info.title`

---

#### Rule 1.2 — Enum values must be UPPER_SNAKE_CASE

Every value in every `enum` array throughout the spec must be UPPER_SNAKE_CASE.

Conversion rules:

- `"mobile"` → `"MOBILE"`
- `"workPhone"` → `"WORK_PHONE"` (split camelCase at word boundaries)
- `"home_phone"` → `"HOME_PHONE"`
- `"OTHER"` → `"OTHER"` (already correct)

**What to check:** All `enum` arrays in `components/schemas`, inline schemas, parameters, and request/response bodies.

---

#### Rule 2.1.1 — URI path parameters must be lowerCamelCase

Every path parameter name must be lowerCamelCase — both in the path template string (e.g. `/contacts/{ContactId}`) and in the corresponding parameter's `name` field.

Conversion rules:

- `{ContactId}` → `{contactId}` (first letter lowercased)
- `{User_Id}` → `{userId}` (remove underscore, camelCase)
- `{CONTACT_ID}` → `{contactId}`

**What to check:**

- Keys in `paths` that contain `{...}` placeholders
- `name` field of parameters with `in: path`
- Any `$ref` targets for path parameters (fix them in `components/parameters` too)

When a path key changes (e.g. `/contacts/{ContactId}` → `/contacts/{contactId}`), update it everywhere it appears.

---

#### Rule 2.2.1 — Descriptions must be non-empty

A `description` field that is an empty string (`""`) or missing entirely is a violation. Every item in the list below must have a meaningful, contextual description.

**What to check:**

- Each operation on each path (GET, POST, PUT, PATCH, DELETE, etc.) — the operation-level `description`
- Each query parameter — `description`
- Each header parameter (in request or response) — `description`
- Each cookie parameter — `description`
- Each `requestBody` — `description` (the `description` field directly on the `requestBody` object)
- Each response object (under `responses`) — `description`
- Each schema property in `components/schemas` — `description`
- Each entry in `components/responses` — `description`
- Each entry in `tags` — `description`

**How to generate descriptions:** Use the field name, type, format, surrounding schema context, and HTTP method to write a short, precise description in plain English (1–2 sentences). Examples:

- A query param named `page` of type integer → "The page number to retrieve in a paginated result set."
- A response with status 200 on a GET /contacts → "A paginated list of contacts matching the query."
- A schema property `createdAt` of format `date-time` → "The timestamp when this record was created, in ISO 8601 format."
- A `BadRequest` response component → "The request was malformed or contained invalid parameters."
- A `Contacts` tag → "Operations for managing contact records."

Do not use placeholder text like "TODO", "Description here", or repeat the field name verbatim.

---

#### Rule 2.3.1 — Integer types must have `minimum` and `maximum`

Every schema with `type: "integer"` must define both `minimum` and `maximum`. If either is missing, add it.

**Sensible defaults by context:**

- Pagination page numbers → `minimum: 1, maximum: 10000`
- Pagination limit/page size → `minimum: 1, maximum: 100`
- Count/total fields → `minimum: 0, maximum: 2147483647`
- Unknown context → `minimum: 0, maximum: 2147483647`

Preserve any existing `minimum`/`maximum` that is already set.

**What to check:** All schemas with `type: "integer"` anywhere in the spec (components, inline schemas, parameters).

---

#### Rule 2.3.2 — String types must have `minLength` and `maxLength`

Every schema with `type: "string"` must define both `minLength` and `maxLength`. If either is missing, add it.

**Exceptions:** Strings with `format: "date-time"`, `format: "date"`, or `format: "time"` are exempt — the format already constrains the value. Strings with a `pattern` that fully constrains length are also exempt.

**Sensible defaults by context:**

- UUID fields (`format: "uuid"`) → `minLength: 36, maxLength: 36`
- Name fields (firstName, lastName, etc.) → `minLength: 1, maxLength: 100`
- Email → `minLength: 5, maxLength: 254`
- Short code/identifier → `minLength: 1, maxLength: 50`
- Free-text/message fields → `minLength: 1, maxLength: 1000`
- Postal codes → `minLength: 3, maxLength: 20`
- Country, city, state → `minLength: 1, maxLength: 100`
- Street address → `minLength: 1, maxLength: 200`
- Unknown context → `minLength: 1, maxLength: 255`

Preserve any existing `minLength`/`maxLength` that is already set.

**What to check:** All schemas with `type: "string"` anywhere in the spec (components, inline schemas, parameters).

---

### Step 3 — Apply all fixes

Derive the output path from the input path by inserting `-fixed-md` before the file extension. Examples:

- `api.yaml` → `api-fixed.yaml`
- `openapi.json` → `openapi-fixed.json`
- `specs/contacts.yaml` → `specs/contacts-fixed.yaml`

Write the full fixed content to the new output path using the Write tool. Do not modify the original file. Preserve all existing valid content exactly as-is — do not change anything that already complies.

If the spec is JSON, write valid JSON. If it is YAML, write valid YAML. Preserve the original formatting style as much as possible (indentation, key ordering, etc.).

### Step 4 — Verify and patch (max 2 retries)

Run the JSON/YAML syntax check against the output file (don't re-read it — use content already in context):

```bash
python3 -c "import json,sys; json.load(sys.stdin)" < <output-path>
```

Then audit the written content against every rule from Step 2:

- All path parameter names match between `paths` keys and `parameters[].name` fields
- No new violations were introduced
- Each rule from Step 2 passes

If any violations or syntax errors remain, fix them with a single rewrite of the output file. Repeat this check-and-fix cycle at most **2 times**. After 2 retries, note any unresolved violations for manual review.

### Step 5 — Report

Print a structured summary to the user:

```
## OpenAPI Compliance Report

**Original file:** <input path>
**Fixed file:** <output path>
**Total violations fixed:** <N>

### Fixes applied

**Rule 1.1 — API Name**
- <what was changed, or "No violations found">

**Rule 1.2 — Enum values**
- <list each enum field and its old → new values, or "No violations found">

**Rule 2.1.1 — URI parameters**
- <list each parameter and the change made, or "No violations found">

**Rule 2.2.1 — Descriptions**
- <count of descriptions added, with a few examples, or "No violations found">

**Rule 2.3.1 — Integer limits**
- <list fields where min/max were added, or "No violations found">

**Rule 2.3.2 — String limits**
- <list fields where minLength/maxLength were added, or "No violations found">

### Compliance verification

**Result:** PASS / FAIL
- <For each rule: "Rule X.Y — PASS" or "Rule X.Y — FAIL: <remaining violation details>">

### Manual review recommended
<List any violations that could not be automatically resolved, or "None">
```
