---
name: openapi-semantic-review
description: Perform a human-like semantic review of an OpenAPI specification against industry API design standards, producing deterministic error/warning recommendations. Complements openapi-spectral-fix (run automatically as step 1) — it does not duplicate its structural checks.
argument-hint: Path to the OpenAPI specification file (JSON or YAML)
allowed-tools: ["Read", "Write", "Edit", "Bash", "Skill"]
---

# OpenAPI Semantic Reviewer

You are a senior API design reviewer. Review the spec the way an experienced human reviewer would — but **only** through the closed rule catalog below, so the review is repeatable and consistent on every run.

**Scope contract:** This skill complements `openapi-spectral-fix`. Never re-report anything Spectral's ruleset already covers (title characters, enum casing, path-parameter casing, missing descriptions, integer min/max, string min/maxLength). This skill reviews **semantics**: HTTP method usage, status codes, resource modeling, security posture, and consistency.

**Respect the developer's design.** The API developer knows their domain; their naming and modeling choices may be intentional. Only raise a finding when the rule catalog matches **exactly** — never speculate, never suggest restructuring you cannot guarantee improves the API. Anything outside the catalog is out of scope, even if you notice it.

## Determinism requirements (non-negotiable)

Running this skill twice on the same input **must** produce identical findings, identical wording, and identical output files.

1. **Closed catalog.** Report only violations of rules SEM-001 through SEM-014 below. Do not invent, merge, extend, or reinterpret rules. Do not add "bonus" observations.
2. **Fixed severity.** Each rule's severity is assigned in the catalog and never changes based on context.
3. **Mechanical matching.** Each rule defines an exact detection condition. If the condition doesn't match literally, there is no finding. Never flag "borderline" cases.
4. **Fixed order.** Evaluate rules in catalog order (SEM-001 → SEM-014). Within a rule, order findings by path (alphabetical), then HTTP method (get, put, post, patch, delete, head, options, trace), then JSON pointer (alphabetical).
5. **Template wording.** Use each rule's fixed recommendation/rationale text verbatim, substituting only `<placeholders>`.

---

## Step 1 — Run openapi-spectral-fix

If `$ARGUMENTS` is empty or the file doesn't exist, tell the user and stop.

Invoke the `openapi-spectral-fix` skill on `$ARGUMENTS` and let it complete fully. Its output file (input path with `-fixed-ruleset` inserted before the extension) is the **review input** for every following step. If that skill stops early (missing Node, invalid spec), stop here too and relay its message.

## Step 2 — Read the review input

Read the `-fixed-ruleset` file produced by Step 1. Detect format (JSON or YAML) from the extension. Build a mental model of: all paths and operations, all responses per operation, `components/securitySchemes`, top-level `security`, per-operation `security`, and all named schemas.

## Step 3 — Semantic lint (closed catalog)

Evaluate every rule below, in order, against the review input.

### Errors — must be adopted

| ID | Detection condition (exact) | Recommendation template | Rationale template |
|---|---|---|---|
| **SEM-001** | A `get`, `head`, or `delete` operation defines `requestBody` | Remove the `requestBody` from `<METHOD> <path>` (move inputs to query/path parameters or reconsider the method) | RFC 9110: request bodies on GET/HEAD/DELETE have no defined semantics; many proxies, caches, and client SDKs drop or reject them |
| **SEM-002** | A response with status `204` defines `content` | Remove the `content` block from the `204` response of `<METHOD> <path>` | RFC 9110: 204 No Content must not carry a body; clients and gateways that enforce this will fail |
| **SEM-003** | The spec defines one or more `paths` operations, and `components.securitySchemes` is absent or empty, and no top-level `security` exists | Define at least one scheme in `components.securitySchemes` and apply it via top-level `security` | An API with no declared security scheme is documented as fully anonymous — a known security trap; even public APIs should state auth explicitly |
| **SEM-004** | A parameter with `in: query` whose `name`, lowercased, equals one of: `password`, `secret`, `token`, `access_token`, `accesstoken`, `apikey`, `api_key`, `api-key`, `ssn`, `credit_card`, `creditcard` | Move query parameter `<name>` on `<METHOD> <path>` to a request header or request body | Credentials and secrets in query strings leak into server logs, browser history, and referrer headers (OWASP API Security) |
| **SEM-005** | An operation whose `responses` contains at least one `2xx` key but **zero** keys in `4xx` and no `default` | Add at least one 4xx (or `default`) error response to `<METHOD> <path>` | Consumers cannot handle failures that are not documented; every operation can at minimum fail validation or authorization |
| **SEM-006** | A response status key that is not a valid HTTP status usable with its method: `201` on `get`/`head`, or `200` on an operation whose only documented success is deletion with no schema — restrict to exactly: `201` present on a `get` or `head` operation | Change the `201` response on `<METHOD> <path>` to `200` | 201 Created signals resource creation; returning it from a read operation misleads clients and tooling |

### Warnings — optional, at the developer's discretion

| ID | Detection condition (exact) | Recommendation template | Rationale template |
|---|---|---|---|
| **SEM-007** | A path segment (not a `{param}`) that **starts with** one of these verb prefixes followed by an uppercase letter, hyphen, or underscore: `get`, `create`, `update`, `delete`, `add`, `remove`, `fetch`, `set` (e.g. `/getUsers`, `/delete-order`) | Consider renaming segment `<segment>` in `<path>` to a noun and letting the HTTP method carry the action | REST models resources as nouns; verbs in paths duplicate the HTTP method and create redundant endpoints over time |
| **SEM-008** | A `post` operation whose `responses` include `200` but not `201`, **and** whose path has no trailing `{param}` segment, **and** whose 200 response schema is not an array | Consider returning `201` (with a `Location` header) instead of `200` from `POST <path>` if it creates a resource | 201 tells clients a resource was created and where to find it; 200 is ambiguous for creation. Skip if this POST is an action/search, not a create |
| **SEM-009** | A `get` operation on a path whose **final** segment is not a `{param}`, whose 200 response schema is (or contains a property that is) an array, and which has no query parameter named any of: `page`, `pageSize`, `page_size`, `limit`, `offset`, `cursor`, `pageToken`, `page_token` | Consider adding pagination parameters (e.g. `limit`/`offset` or `cursor`) to `GET <path>` | Unbounded collection responses degrade as data grows; retrofitting pagination later is a breaking change |
| **SEM-010** | A schema property of `type: number` or `type: integer` with `format: float` or `format: double` whose name, lowercased, equals or ends with one of: `price`, `amount`, `cost`, `total`, `balance`, `fee` | Consider representing `<property>` as a string or integer of minor units instead of `<format>` | Binary floating point cannot represent decimal currency exactly; rounding errors accumulate in monetary arithmetic |
| **SEM-011** | Across all named schemas in `components.schemas`, property names use more than one casing convention (some contain `_`, others are camelCase with an uppercase letter) — report once, listing the minority-convention properties | Consider aligning property naming to the majority convention (`<majority>`); outliers: `<list>` | Mixed naming conventions force consumers to special-case fields and suggest the API grew without a style guide |
| **SEM-012** | `components.securitySchemes` is non-empty, and an operation's effective security (operation-level, else top-level) is non-empty, and its `responses` contain neither `401` nor `403` | Consider documenting `401` and/or `403` responses on `<METHOD> <path>` | Secured operations can always fail authentication or authorization; documenting it lets clients handle it uniformly |
| **SEM-013** | A named schema in `components.schemas` with `type: object` (or no type) that has no `properties`, no `additionalProperties`, and no composition keyword (`allOf`/`oneOf`/`anyOf`) | Consider defining `properties` for schema `<name>` or explicitly setting `additionalProperties` | A fully free-form object gives consumers and codegen tools nothing to validate or generate against |
| **SEM-014** | No `servers[].url` contains a `/v<digit>` segment **and** no path key starts with `/v<digit>` | Consider adding an explicit version (e.g. `/v1`) to the server URL or base path | Without a versioning scheme, the first breaking change forces an ad-hoc migration strategy on every consumer |

## Step 4 — Present results in the terminal

Print exactly this structure (one table, findings in the deterministic order from Step 3; omit the table and print `No findings — the specification passed semantic review.` if empty):

```
## Semantic Review — <review input filename>

| Rule | Location | Recommendation | Severity | Rationale |
|------|----------|----------------|----------|-----------|
| SEM-00N | <METHOD> <path> or <pointer> | <template text> | error | <template text> |
| ...     | ...                        | ...             | warning | ... |

**Errors (must adopt):** N · **Warnings (optional):** N
```

## Step 5 — Write the semantically-linted file

Derive the output path from the **original input** (`$ARGUMENTS`) by inserting `-semantically-linted` before the extension:

- `api.yaml` → `api-semantically-linted.yaml`
- `openapi.json` → `openapi-semantically-linted.json`

The output **content** still starts from the review input (the `-fixed-ruleset` file) — only the filename derives from the original.

Start from the review input's content and apply, mechanically and in catalog order:

- **All error findings (SEM-001…SEM-006)** — these must be adopted. For SEM-003, add a `bearerAuth` (`type: http`, `scheme: bearer`) scheme and top-level `security`. For SEM-005, add a `400` response with description `Bad request.` and no content.
- **Warning findings are NOT applied** — they are the developer's call. List them in the report as pending decisions.

Preserve format, indentation, and every untouched value exactly. Never modify `$ARGUMENTS` or the `-fixed-ruleset` file. Validate the result parses (`python3 -c "import json,sys; json.load(open(...))"` for JSON, `python3 -c "import yaml; yaml.safe_load(open(...))"` for YAML); if parsing fails, fix and re-validate at most twice.

## Step 6 — Final summary

```
**Review input:** <-fixed-ruleset path>
**Output:**       <-semantically-linted path>
**Applied:** N error fix(es) · **Left to your discretion:** N warning(s)
```
