---
name: CLI MCP nomenclature audit
overview: A focused research pass to map argument and type naming across RuVector MCP tool schemas, dispatch handlers, shared wrappers, and Commander CLI flags—surfacing mismatches (e.g. dimension vs dimensions, string vs number/BigInt for IDs), schema vs runtime truth, and whether errors guide users to a correct second attempt.
todos:
  - id: schema-extract
    content: Tabulate MCP inputSchema keys and JSON types per tool (group by rvf_, hooks_, workers_, …)
    status: completed
  - id: handler-map
    content: "Trace mcp-server.js switch handlers: args → require() calls; note renames vs schema"
    status: completed
  - id: cli-map
    content: Map CLI flags/positionals for same domains to compare naming to MCP
    status: completed
  - id: library-contract
    content: Record @ruvector/rvf + rvf-wrapper canonical option/entry shapes
    status: completed
  - id: error-review
    content: Sample failure responses; rate whether errors teach correct keys/types for retry
    status: completed
  - id: write-nomenclature-doc
    content: Add ../analysis/ruvector/CLI_MCP_NOMENCLATURE_RESEARCH.md + README index row
    status: completed
isProject: false
---

# CLI / MCP nomenclature and ergonomics — research plan

## Goal

Produce a **nomenclature and type-alignment audit** (same style as [CLI_MCP_ECOSYSTEM_RESEARCH.md](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) that answers:

1. Do **keys** align across MCP `inputSchema`, handler `args`, shared modules (`rvf-wrapper`, `@ruvector/rvf`), and CLI flags/positional names?
2. Do **value types** align (JSON Schema `number` vs `string`, arrays of `number` for IDs vs string labels, BigInt in WASM paths)?
3. On failure, do **errors + hints** give enough information to fix the call on a second attempt (missing required field, wrong key name, wrong type)?

## Scope

- **In scope:** [`npm/packages/ruvector/bin/mcp-server.js`](npm/packages/ruvector/bin/mcp-server.js) (every tool `inputSchema` + `case` handler args), [`npm/packages/ruvector/bin/cli.js`](npm/packages/ruvector/bin/cli.js) (flags and arguments for the same domains), [`npm/packages/ruvector/src/core/rvf-wrapper.js`](npm/packages/ruvector/src/core/rvf-wrapper.js), and downstream [`npm/packages/rvf`](npm/packages/rvf) public types (`RvfOptions`, ingest/query/delete shapes).
- **Out of scope (unless referenced):** Rust `rvf-cli` flags; other npm packages’ MCP servers.

## Methods

- **Inventory:** Extract per-tool rows: MCP tool name, each `inputSchema.properties` key + `type`/`items`, handler mapping (what is passed to `require(...)` functions), CLI equivalent flag long/short name and positional names.
- **Diff pass:** Mark **exact match**, **alias risk** (same meaning, different name), **type mismatch** (schema says X, runtime expects Y), **undocumented** (schema too loose, e.g. `items: { type: 'object' }` without shape).
- **Error sampling:** For representative tools (RVF create/ingest/delete/query, hooks_route, workers_dispatch, one brain tool), trace what is returned on validation failure vs thrown errors—whether `hint`, `message`, or JSON includes **actionable** key names.

## Already observed high-priority discrepancy (seed for audit)

**RVF create — `dimension` vs `dimensions`**

- MCP schema and tool copy use **`dimension`** ([`mcp-server.js` ~1126](npm/packages/ruvector/bin/mcp-server.js)).
- MCP handler calls `createRvfStore(safePath, { dimension: args.dimension, ... })` ([~3006](npm/packages/ruvector/bin/mcp-server.js)).
- [`RvfOptions`](npm/packages/rvf/src/types.ts) and [`RvfDatabase.create`](npm/packages/rvf/src/database.ts) expect **`dimensions`** (plural).
- CLI uses **`dimensions: opts.dimension`** from `-d, --dimension` ([`cli.js` ~7064](npm/packages/ruvector/bin/cli.js)).

This is a textbook **key mismatch** between MCP handler and `@ruvector/rvf` API (CLI path is closer to the library). The audit should confirm runtime behavior (failure vs silent wrong default) and recommend **one canonical name** plus optional **aliases** in the handler.

**RVF delete — ID types**

- MCP `rvf_delete` schema declares **`ids` as array of `number`** ([~1175](npm/packages/ruvector/bin/mcp-server.js)).
- [`@ruvector/rvf` Node path](npm/packages/rvf/src/backend.ts) uses **string IDs** with numeric label mapping; ingest `entries[].id` is string-oriented in TS types.

Research should document whether MCP clients sending strings fail schema validation vs fail at runtime, and align schema + docs with actual `delete` contract.

## Deliverable document (when executed)

Suggested path: [`../analysis/ruvector/CLI_MCP_NOMENCLATURE_RESEARCH.md`](../analysis/ruvector/CLI_MCP_NOMENCLATURE_RESEARCH.md), structured like the ecosystem doc:

1. **Executive summary** — top 10 naming/type mismatches.
2. **Per-domain tables** — RVF, workers, hooks, rvlite, brain, edge, identity, decompile: columns `MCP_key`, `MCP_schema_type`, `handler_mapping`, `CLI_flag_or_arg`, `library_key`, `match_status`, `error_quality_notes`.
3. **Cross-cutting conventions** — snake_case MCP vs kebab-case CLI; `path` vs `db_path`; `parent_path`/`child_path` vs positional args.
4. **Ergonomic recommendations** — e.g. JSON Schema `oneOf` for string|number IDs where supported; **machine-readable error payloads** listing `expectedKeys`, `receivedKeys`; README table “MCP arg → CLI equivalent”.
5. **Epics** — align `rvf_create` options; fix `rvf_delete` schema; add integration tests that assert error bodies contain corrective hints; optional **schema codegen** from TypeScript types for `@ruvector/rvf`.

## Suggested execution todos

- **schema-extract** — Parse or manually tabulate all MCP `inputSchema` blocks grouped by tool prefix (`rvf_`, `hooks_`, …).
- **handler-map** — For each prefix, trace `args` usage in the big `switch` and note renames/omissions.
- **cli-map** — For the same capabilities, extract Commander option names and positionals from `cli.js`.
- **library-contract** — Read `@ruvector/rvf` and wrapper typings; record canonical field names.
- **error-review** — Sample 8–12 failure modes; score hints (none / generic / actionable).
- **write-doc** — Author `CLI_MCP_NOMENCLATURE_RESEARCH.md` and add a row to [`analysis/ruvector/README.md`](../analysis/ruvector/README.md).

## Non-goals (this research phase)

- Implementing fixes (that belongs to follow-up PRs after sign-off).
- Changing MCP SDK or Commander.
