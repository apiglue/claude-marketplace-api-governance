# claude-marketplace-api-governance

A [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces) providing API governance and design tooling.

## Installing the marketplace

```
/plugin marketplace add apiglue/claude-marketplace-api-governance
```

Then install plugins from it with `/plugin install <plugin-name>@apiglue-api-governance`.

## Plugins

### api-ai-toolkit

An API toolkit focused on API design — audits and fixes OpenAPI specifications so they comply with a shared set of RESTful API design standards.

Install: `/plugin install api-ai-toolkit@apiglue-api-governance`

#### Skills

- **`openapi-md-fix`** — Audits an OpenAPI spec against the RESTful design standards directly (no external linter) and writes a fixed copy. Covers: alphanumeric `info.title`, UPPER_SNAKE_CASE enum values, lowerCamelCase path parameters, non-empty descriptions across operations/parameters/schemas/responses/tags, `minimum`/`maximum` on integer schemas, and `minLength`/`maxLength` on string schemas. Produces a compliance report and writes output to `<name>-fixed.<ext>`.

  Usage: `/openapi-md-fix path/to/spec.yaml`

- **`openapi-spectral-fix`** — Lints an OpenAPI spec with [Spectral](https://github.com/stoplightio/spectral) using the bundled `api-design-standards` ruleset, then iteratively fixes reported errors (and optionally warnings) until the spec passes, up to 3 iterations. Requires Node.js ≥ 22. Writes output to `<name>-fixed-ruleset.<ext>` without touching the original file.

  Usage: `/openapi-spectral-fix path/to/spec.yaml`

#### Config

- **`config/api-design-standards-ruleset.json`** — Spectral ruleset encoding the RESTful API design standards (naming conventions, required descriptions, enum casing, numeric/string bounds) used by `openapi-spectral-fix`.
