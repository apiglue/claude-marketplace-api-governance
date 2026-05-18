---
name: openapi-spectral-fix
description: Lint an OpenAPI specification with Spectral and iteratively fix all errors (and optionally warnings) using the api-design-standards ruleset
argument-hint: Path to the OpenAPI specification file (JSON or YAML)
allowed-tools: ["Read", "Write", "Edit", "Bash"]
---

# OpenAPI Spectral Fixer

You are an expert API designer. Your task is to lint the provided OpenAPI specification using Spectral and iteratively fix all errors (and optionally warnings) reported by the linter.

## Inputs

- OpenAPI spec to fix: `$ARGUMENTS`
- Spectral ruleset: `./config/api-design-standards-ruleset.json` (relative to the current working directory)

---

## Process

### Step 1 — Validate input

If `$ARGUMENTS` is empty or not provided, tell the user:
> Please provide a path to an OpenAPI specification file. Example: `/openapi-spectral-fix path/to/openapi.yaml`

Then stop.

Read the file at `$ARGUMENTS`. If it does not exist, tell the user and stop.

Confirm it is a valid OpenAPI specification by verifying the presence of:
- A top-level `openapi` key (OpenAPI 3.x) or `swagger` key (Swagger 2.x)
- An `info` object containing at least `title` and `version`
- A `paths` object

If any of these are missing, tell the user exactly what is missing and stop. Do not proceed with an invalid spec.

---

### Step 2 — Ensure dependencies

**Check Node.js:**

```bash
node --version 2>&1
```

If Node.js is not installed or the version is below v22, print:
> Node.js 22 or higher is required. Please install it from https://nodejs.org and try again.

Then stop.

**Install Spectral CLI if needed:**

```bash
npx @stoplight/spectral-cli --version 2>/dev/null || npm install -g @stoplight/spectral-cli
```

Confirm Spectral is available before continuing.

---

### Step 3 — Determine output file path

Derive the output path from the input path by inserting `-fixed` before the file extension:

- `api.yaml` → `api-fixed.yaml`
- `openapi.json` → `openapi-fixed.json`
- `specs/contacts.yaml` → `specs/contacts-fixed.yaml`

**Never modify the original file.** Copy the original content to the output path now. All fixes in subsequent steps apply only to the output file.

---

### Step 4 — Initial lint

Run Spectral against the **original** input file and capture the output:

```bash
npx @stoplight/spectral-cli lint "$ARGUMENTS" --ruleset ./config/api-design-standards-ruleset.json --format json 2>&1 || true
```

Parse the JSON array. Each result object has:
- `code` — rule ID (e.g. `RESTFUL-STD-1.1-NAMING-FUNDAMENTALS-API-NAME`)
- `message` — human-readable violation description
- `severity` — `0` = error, `1` = warning, `2` = info, `3` = hint
- `path` — array of JSON-path segments pointing to the offending value
- `range` — `{ start: { line, character }, end: { line, character } }` in the source file

Separate results into:
- **Errors** (severity 0): must be fixed before asking about warnings
- **Warnings** (severity 1): only fix after errors are resolved, and only if the user confirms
- **Info / Hint** (severity 2–3): ignore

If there are zero errors and zero warnings, skip to Step 7 and report the spec is already fully compliant.

Store the total initial error count and warning count — you will need them for the final report.

---

### Step 5 — Fix errors (max 5 iterations)

Iterate up to **5 times** to resolve all errors. Track the current iteration number starting at 1.

**Each iteration:**

1. Read the current output file.
2. For every error in the latest lint results, apply the appropriate fix based on the rule ID:

   | Rule | Fix |
   |------|-----|
   | `RESTFUL-STD-1.1-NAMING-FUNDAMENTALS-API-NAME` | Remove all non-alphanumeric/non-space characters from `info.title` |
   | `RESTFUL-STD-1.2-ENUM-UPPER-SNAKE-CASE` | Convert each violating enum value to UPPER_SNAKE_CASE (split camelCase at word boundaries, replace non-word chars with `_`, uppercase all) |
   | `RESTFUL-STD-2.1.1-CONSISTENCY-FUNDAMENTALS-URI-PARAMS` | Rename the path parameter to lowerCamelCase in both the path template string and the parameter's `name` field |
   | `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-PATH-PARAMS` | Add a concise, contextual `description` to the path parameter |
   | `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-QUERY-PARAMS` | Add a concise, contextual `description` to the query parameter |
   | `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-REQUEST-HEADERS` | Add a concise, contextual `description` to the request header parameter |
   | `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-RESPONSE-HEADERS` | Add a concise, contextual `description` to the response header |
   | `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-REQUEST-BODY` | Add a concise, contextual `description` to the request body |
   | `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-RESPONSE-BODY` | Add a concise, contextual `description` to the response object |
   | `RESTFUL-STD-2.1.2-CONSISTENCY-FUNDAMENTALS-DESCRIPTION-COOKIES` | Add a concise, contextual `description` to the cookie |
   | `RESTFUL-STD-2.3.1-CONSISTENCY-FUNDAMENTALS-INTEGER-MAX` | Add `maximum` to the integer parameter schema (use context-appropriate default: pagination page → 10000, page size → 100, otherwise → 2147483647) |
   | `RESTFUL-STD-2.3.1-CONSISTENCY-FUNDAMENTALS-INTEGER-MIN` | Add `minimum` to the integer parameter schema (pagination → 1, otherwise → 0) |
   | `RESTFUL-STD-2.3.2-CONSISTENCY-FUNDAMENTALS-STRING-MAXLENGTH` | Add `maxLength` to the string parameter schema (email → 254, name → 100, address → 200, uuid → 36, otherwise → 255). |
   | `RESTFUL-STD-2.3.2-CONSISTENCY-FUNDAMENTALS-STRING-MINLENGTH` | Add `minLength` to the string parameter schema (uuid → 36, email → 5, otherwise → 1). |

   When writing descriptions, make them precise and contextual — use the parameter name, type, HTTP method, and surrounding schema. Never use placeholder text like "TODO" or "Description here".

3. Write all fixes to the output file. Preserve the original file format (JSON or YAML) and existing indentation style. Do not alter any value that already complies.

4. Re-run Spectral against the **output file**:

   ```bash
   npx @stoplight/spectral-cli lint "<output-file>" --ruleset ./config/api-design-standards-ruleset.json --format json 2>&1 || true
   ```

5. Count remaining errors (severity 0):
   - If **0 errors remain**: exit the loop and proceed to Step 6.
   - If errors remain and this is **iteration < 5**: increment the iteration counter and repeat from step 1 of this loop.
   - If errors remain and this is **iteration 5**: stop the loop. Note these unresolved errors — you will include them in the final report and inform the user.

---

### Step 6 — Fix warnings (optional, after errors are resolved)

After the error-fix loop ends (whether all errors were resolved or max iterations was reached), check if any **warnings** (severity 1) remain in the last lint result.

If there are warnings, ask the user:
> There are **N warning(s)** in the fixed spec. Would you like me to fix those as well?

Wait for the user's response.
- If **yes**: apply the same iterative fix process (max 5 more iterations) targeting only warnings (severity 1). Use the same fix logic from Step 5, applying it to warnings.
- If **no** or any other response: skip warning fixes and proceed to Step 7.

If there are no warnings, proceed directly to Step 7.

---

### Step 7 — Final report

Print a structured summary:

```
## OpenAPI Spectral Fix Report

**Original file:** <input path>
**Fixed file:**    <output path>

### Lint results

| Severity | Initial | Remaining |
|----------|---------|-----------|
| Errors   | N       | N         |
| Warnings | N       | N         |

### Fixes applied

<For each rule that had at least one violation, list:>
**<rule-id>**
- <what was changed; for bulk changes like descriptions, give count + a few examples>

### Unresolved issues
<List any errors or warnings that could not be automatically fixed after max iterations, with their rule ID, path, and message. If none, write "None".>

### Result
PASS — all errors resolved.
  — OR —
PARTIAL — N error(s) remain after 5 fix iterations. Manual review required.
```
