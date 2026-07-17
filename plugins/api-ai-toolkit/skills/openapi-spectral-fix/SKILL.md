---
name: openapi-spectral-fix
description: Lint an OpenAPI specification with Spectral and iteratively fix all errors (and optionally warnings) using the api-design-standards ruleset
argument-hint: Path to the OpenAPI specification file (JSON or YAML)
allowed-tools: ["Read", "Write", "Edit", "Bash"]
---

# OpenAPI Spectral Fixer

Fix all Spectral lint errors in an OpenAPI spec using the `api-design-standards` ruleset.

**Inputs:** spec at `$ARGUMENTS` · ruleset at `${CLAUDE_PLUGIN_ROOT}/config/api-design-standards-ruleset.json`

---

### Step 1 — Validate input

If `$ARGUMENTS` is empty or the file doesn't exist, tell the user and stop.

Verify the spec has `openapi` or `swagger`, `info.title`, `info.version`, and `paths`. If any are missing, report what's missing and stop.

---

### Step 2 — Ensure dependencies

Run the following single command — require Node.js ≥ 22 and confirm Spectral is available:

```bash
node --version && npx --yes @stoplight/spectral-cli --version
```

Stop if Node is missing or below v22. Stop if Spectral fails to load.

---

### Step 3 — Output path

Insert `-fixed-ruleset` before the extension (`api.yaml` → `api-fixed-ruleset.yaml`). Copy the original to the output path now — never modify the original.

Read the output path file into context now. From this point on, every fix in Step 5 and Step 6 must be read from and written to the **output path only** — never touch `$ARGUMENTS`.

---

### Step 4 — Initial lint

```bash
npx --yes @stoplight/spectral-cli lint "$ARGUMENTS" --ruleset "${CLAUDE_PLUGIN_ROOT}/config/api-design-standards-ruleset.json" --format json 2>&1 || true
```

Parse the JSON array. Relevant fields: `code`, `message`, `severity` (0=error, 1=warning, 2–3=ignore), `path`.

Record initial error and warning counts. If both are zero, skip to Step 7.

---

### Step 5 — Fix errors (max 3 iterations)

Repeat until 0 errors remain or 3 iterations exhausted:

1. Using the output path file's current content (already in context — never the original at `$ARGUMENTS`), apply fixes for **every violation** in the lint output in a single write operation — process the complete list before writing.
2. Write the result to the output path (preserve format and indentation; don't touch compliant values). Never write to `$ARGUMENTS`.
3. Re-lint the output file: same command as Step 4 but targeting the output path.
4. If 0 errors: exit loop. If iteration 3 and errors remain: note them, exit loop.

**Rule table:**

| Rule code                                                                 | Fix                                                                                                                                    |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `RESTFUL-STD-1.1-NAMING-FUNDAMENTALS-API-NAME`                            | Strip non-alphanumeric/non-space chars from `info.title`                                                                               |
| `RESTFUL-STD-1.2-ENUM-UPPER-SNAKE-CASE`                                   | Convert violating enum values to UPPER*SNAKE_CASE (split camelCase at word boundaries, replace non-word chars with `*`, uppercase all) |
| `RESTFUL-STD-2.1.1-CONSISTENCY-FUNDAMENTALS-URI-PARAMS`                   | Rename path parameter to lowerCamelCase in both the path template string and the parameter's `name` field                              |
| `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-PATH-PARAMS`      | Add a concise contextual `description` to the path parameter                                                                           |
| `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-QUERY-PARAMS`     | Add a concise contextual `description` to the query parameter                                                                          |
| `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-REQUEST-HEADERS`  | Add a concise contextual `description` to the request header                                                                           |
| `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-RESPONSE-HEADERS` | Add a concise contextual `description` to the response header                                                                          |
| `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-REQUEST-BODY`     | Add a concise contextual `description` to the request body                                                                             |
| `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-RESPONSE-BODY`    | Add a concise contextual `description` to the response object                                                                          |
| `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-COOKIES`          | Add a concise contextual `description` to the cookie parameter                                                                         |
| `RESTFUL-STD-2.3.1-CONSISTENCY-FUNDAMENTALS-INTEGER-MAX`                  | Add `maximum` (pagination page → 10000, page size → 100, else → 2147483647)                                                            |
| `RESTFUL-STD-2.3.1-CONSISTENCY-FUNDAMENTALS-INTEGER-MIN`                  | Add `minimum` (pagination → 1, else → 0)                                                                                               |
| `RESTFUL-STD-2.3.2-CONSISTENCY-FUNDAMENTALS-STRING-MAXLENGTH`             | Add `maxLength` (email → 254, name → 100, address → 200, uuid → 36, else → 255)                                                        |
| `RESTFUL-STD-2.3.2-CONSISTENCY-FUNDAMENTALS-STRING-MINLENGTH`             | Add `minLength` (uuid → 36, email → 5, else → 1)                                                                                       |

Descriptions must be precise and contextual — use the parameter name, type, HTTP method, and schema. Never use placeholder text.

---

### Step 6 — Fix warnings (optional)

If warnings remain after Step 5, ask:

> There are **N warning(s)**. Fix them too?

- Yes → same iterative process (max 3 iterations), severity 1 only.
- No → proceed to Step 7.

---

### Step 7 — Report

```
## OpenAPI Spectral Fix Report

**Input:**  <input path>
**Output:** <output path>

| Severity | Initial | Remaining |
|----------|---------|-----------|
| Errors   | N       | N         |
| Warnings | N       | N         |

### Fixes applied
<Per rule with violations: rule-id → what changed (count + examples for bulk changes)>

### Unresolved
<Rule ID, path, message for each unresolved issue — or "None">

**Result:** PASS — all errors resolved.
  — OR —
**Result:** PARTIAL — N error(s) remain. Manual review required.
```
