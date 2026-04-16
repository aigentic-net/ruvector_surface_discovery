# ADR-A01: MCP/CLI Surface Alignment — Contract Fixes, Parity, and Manifest

**Status**: Proposed
**Date**: 2026-04-10
**Authors**: clawdette (research + analysis), for review by ruv.io
**Deciders**: ruv.io, Architecture Review Board
**SDK**: Claude-Flow
**Relates to**: ADR-029 (RVF canonical format), ADR-001 (core architecture)

## Context

### The Surface Fragmentation Problem

The `npm/packages/ruvector` package exposes two parallel interfaces to the same underlying capabilities:

- **MCP server** (`bin/mcp-server.js`) — 95 tools across 9 domains (rvf, hooks, workers, rvlite, brain, edge, identity, decompile, learning), consumed by AI agents via stdio
- **Commander CLI** (`bin/cli.js`) — human/script interface covering vector DB, hooks, workers, RVF file ops, diagnostics

Neither surface is a subset of the other. A [full research audit](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md) and [nomenclature deep-dive](../analysis/ruvector/CLI_MCP_NOMENCLATURE_RESEARCH.md) identified three categories of problems:

#### 1. Contract bugs — agents receive wrong results silently


| Tool               | Bug                                                                                       | Impact                                                                       |
| ------------------ | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `rvf_create`       | MCP schema uses `dimension` (singular); `@ruvector/rvf` API expects `dimensions` (plural) | Store created with wrong/default dimension; silent data corruption on ingest |
| `rvf_delete`       | Schema declares `ids` as `number[]`; `RvfDatabase.delete()` expects `string[]`            | Strict MCP clients may fail validation; permissive ones send wrong types     |
| `workers_dispatch` | Returns `success: true` even when subprocess fails                                        | Agents believe task succeeded; no stderr, exit code, or retry hint           |


#### 2. Parity gaps — capability exists on one side only


| Capability             | MCP                                            | CLI                                 | Risk                                      |
| ---------------------- | ---------------------------------------------- | ----------------------------------- | ----------------------------------------- |
| Delete vectors         | `rvf_delete`                                   | No `rvf delete` command             | Humans cannot mirror agent operations     |
| Export RVF metadata    | No `rvf_export` tool                           | `rvf export` command                | Agents cannot export without shelling out |
| Cancel/cleanup workers | No tools                                       | `workers cancel`, `workers cleanup` | Agents cannot manage worker lifecycle     |
| Rvlite queries         | `rvlite_sql`, `rvlite_cypher`, `rvlite_sparql` | No CLI equivalent                   | Scripts cannot query rvlite without MCP   |


#### 3. Structural drift — no shared source of truth

- Tool definitions live only in `mcp-server.js` (hand-maintained `inputSchema` objects in a 3,700+ line switch dispatch)
- CLI flags are defined independently in `cli.js` (Commander options)
- No manifest, no codegen, no parity test — drift is guaranteed over time
- `IntelligenceEngine` constructed eagerly at MCP startup even when only non-hooks tools are called
- Path validation (`validateRvfPath`) exists only in MCP; CLI uses raw user paths

### Research Sources


| Document                                                                                  | Scope                                                                                     |
| ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| [CLI_MCP_ECOSYSTEM_RESEARCH.md](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)       | Inventory, parity matrix, golden paths, test gaps, optimization hypotheses, epic ordering |
| [CLI_MCP_NOMENCLATURE_RESEARCH.md](../analysis/ruvector/CLI_MCP_NOMENCLATURE_RESEARCH.md) | Per-tool schema/handler/CLI/library key alignment, error ergonomics, type mismatches      |


## Decision

### Adopt a phased alignment model with contract correctness first

Fix **silent failures** before adding features. Ship in waves, each producing a testable, releasable increment. One coordinated npm version bump per wave (see [Versioning](#versioning) below).

### Wave 0 — Contract bugs (P0, prerequisite for all other waves)

**Scope:** `bin/mcp-server.js` handler fixes + integration tests.


| Fix                        | Change                                                                                                                              | Acceptance                                                                                 |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `rvf_create` dimensions    | Map `dimension` and `dimensions` to `RvfOptions.dimensions`; accept both in schema for backward compatibility                       | `rvf_create` with `dimension: 384` produces a store that accepts 384-dim vectors on ingest |
| `rvf_delete` IDs           | Change `ids` schema to `string[]` (or `oneOf` string/number with coercion); coerce to `string[]` in handler before `store.delete()` | Integration test: ingest, query, delete with string IDs from results, verify count         |
| `workers_dispatch` failure | Return `{ success: false, stderr, exitCode, hint }` on subprocess failure                                                           | Golden test: dispatch nonexistent command, assert `success: false` + hint                  |
| `rvf-wrapper` review       | Confirm `src/core/rvf-wrapper.ts` passes `dimensions` correctly to `createRvfStore`                                                 | No silent rename or drop of the field in the wrapper layer                                 |


**Files touched:** `[bin/mcp-server.js](../../npm/packages/ruvector/bin/mcp-server.js)` (~lines 1126, 1175, 2731, 3002, 3052), `[src/core/rvf-wrapper.ts](../../npm/packages/ruvector/src/core/rvf-wrapper.ts)`, `[test/](../../npm/packages/ruvector/test/)`.

### Wave 1 — Feature parity (E1 + E2)


| Epic                          | Change                                                                        | Files                                   |
| ----------------------------- | ----------------------------------------------------------------------------- | --------------------------------------- |
| **E1: CLI `rvf delete`**      | Add `ruvector rvf delete <path> --ids <id,...>` mirroring fixed MCP semantics | `bin/cli.js`, `src/core/rvf-wrapper.ts` |
| **E1: MCP `rvf_export`**      | Add `rvf_export` tool OR document explicit workaround (CLI export + script)   | `bin/mcp-server.js` or README           |
| **E2: MCP workers lifecycle** | Add `workers_cancel` and `workers_cleanup` tools mirroring CLI                | `bin/mcp-server.js`                     |


### Wave 2 — Schema hardening + security (P1 + E4)


| Item                         | Change                                                                                                                                                             |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `rvf_ingest` entries schema  | Tighten `items` from `{ type: 'object' }` to `RvfIngestEntry` shape (`id`, `vector`, optional `metadata`)                                                          |
| Structured error envelope    | Add `{ code, field, expectedKeys }` fields on server-side validation failures                                                                                      |
| Error contract tests         | Golden assertions: missing package hint, validation field/expectedKeys, workers_dispatch failure shape                                                             |
| MCP smoke test               | Spawn `node bin/mcp-server.js` with mocked stdio; assert `ListTools` returns expected count + one `CallTool` per critical group (`rvf_create` temp, `hooks_stats`) |
| Path policy (E4)             | Extract `validateRvfPath` to shared module; CLI gets optional `--allow-outside-cwd`; MCP remains cwd-locked                                                        |
| Security review              | Short threat-model note for workers shell exec surface (worker ID format validation, PID targeting) + path traversal (symlinks, null bytes)                        |
| `hooks_init` parity decision | Decide: add MCP `inputSchema` properties for `--minimal`/`--fast` OR document as intentionally CLI-only                                                            |
| Loose `object` schema audit  | Inventory MCP tools with untyped nested `object` payloads (hooks learning); tighten JSON Schema or document as opaque                                              |


### Wave 3 — Manifest + optional codegen (E3)


| Item                       | Change                                                                                                                       |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Single tool manifest       | JSON/TS file: `toolId`, `inputSchema`, `cliEquivalent`, `handler` — generates MCP `tools/list` + `ruvector mcp tools` output |
| Split `mcp-server.js`      | Domain-based handler modules (`handlers/rvf.js`, `handlers/hooks.js`, ...); lazy `require` per group                         |
| Defer `IntelligenceEngine` | Construct on first hooks tool call, not at startup                                                                           |
| Parity checklist test      | Snapshot test from manifest: tool count, new tool presence, schema shape                                                     |


**Performance acceptance criteria:**


| Metric                                     | Target                    |
| ------------------------------------------ | ------------------------- |
| Cold `list_tools` latency                  | <200ms (stdio round-trip) |
| Startup RSS (idle)                         | <50MB                     |
| First `hooks_stats` (triggers engine init) | <500ms                    |


### Wave 4 — Unified documentation


| Item                    | Deliverable                                                                                                           |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Parity surface matrix   | README table: every user-facing capability x MCP x CLI x notes                                                        |
| Conventions doc         | `snake_case` MCP vs kebab CLI; path vocabulary (`path` vs `db_path` vs `parent_path`); camelCase islands (`workerId`) |
| CLI-only / MCP-only     | Explicit lists: hooks lifecycle events (CLI-only), brain tools (MCP-only), `native run` (CLI-only)                    |
| `MCP_SERVER=1` behavior | Document interaction with worker parallelism                                                                          |


### Wave 5+ — Deferred / product-gated

- **E5:** Thin `ruvector rvlite` CLI or document "MCP-only" for `rvlite_sql` / `rvlite_cypher` / `rvlite_sparql`
- **RvfOptions MCP parity:** Expose `ef_search`, `filter`, `m`, `compression` on `rvf_create`/`rvf_query` if product wants feature parity (not just bug fixes); if expanded, evaluate matching CLI `rvf create` flags (`-d`/`-m` only today)
- **Product fork:** Dedicated MCP tools for vector DB `insert`/`search` (currently CLI-only `create`/`insert`/`search`)
- **Low-severity aliases:** `hooks_recall.top_k` vs `topK`; `workerId` vs `worker_id` — breaking MCP revision, separate epic
- **Optional codegen:** Generate `inputSchema` from `@ruvector/rvf` TypeScript types to prevent drift
- `**rvf_examples` filter parity:** Optional CLI `--filter` to match MCP `filter` parameter (research §2.1)

### Versioning


| Wave | Nature                                 | Semver                                                                        |
| ---- | -------------------------------------- | ----------------------------------------------------------------------------- |
| W0   | Breaking (`rvf_delete` schema) + fixes | Minor or major (assess client impact); `oneOf` deprecation period recommended |
| W1   | Additive (new tools, new CLI command)  | Minor                                                                         |
| W2+  | Schema tightening, structured errors   | Patch or minor per scope                                                      |


One CHANGELOG section per wave. First W0 PR opens the draft; subsequent PRs append.

## Consequences

### Positive

- Agents get correct `rvf_create` behavior and honest `workers_dispatch` failures immediately (W0)
- Full RVF lifecycle (create-ingest-query-delete-export) available on both surfaces after W1
- Single manifest eliminates tool drift and enables automated parity testing (W3)
- Lazy `IntelligenceEngine` + domain splitting improves cold-start for non-hooks MCP consumers (W3)

### Negative

- `rvf_delete` `ids` type change is breaking for strict MCP clients that send `number[]` — mitigated by `oneOf` + coercion + deprecation period
- Manifest + domain split is significant refactoring of a 3,700+ line file — risk of introducing regressions; mitigated by MCP smoke tests (W2) before manifest work (W3)
- Path policy shared module adds a new cross-concern import; must not weaken MCP's strict posture when CLI opts in

### Risks


| Risk                                | Mitigation                                                                             |
| ----------------------------------- | -------------------------------------------------------------------------------------- |
| W0 breaks existing MCP integrations | `oneOf` string/number for ids; coerce in handler; deprecation note in CHANGELOG        |
| W3 refactor introduces regressions  | MCP smoke test + path traversal test land in W2 before refactor                        |
| Workers MCP shell exec surface      | Security review: validate worker ID format, confirm kill targets only known child PIDs |


## Key Files


| Area                        | File                                                                                                                                                                                           |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MCP tool schemas + dispatch | `[npm/packages/ruvector/bin/mcp-server.js](../../npm/packages/ruvector/bin/mcp-server.js)`                                                                                                     |
| CLI commands                | `[npm/packages/ruvector/bin/cli.js](../../npm/packages/ruvector/bin/cli.js)`                                                                                                                   |
| RVF wrapper (shared)        | `[npm/packages/ruvector/src/core/rvf-wrapper.ts](../../npm/packages/ruvector/src/core/rvf-wrapper.ts)`                                                                                         |
| Canonical RVF types         | `[npm/packages/rvf/src/types.ts](../../npm/packages/rvf/src/types.ts)`, `[src/backend.ts](../../npm/packages/rvf/src/backend.ts)`, `[src/database.ts](../../npm/packages/rvf/src/database.ts)` |
| Tests                       | `[npm/packages/ruvector/test/](../../npm/packages/ruvector/test/)`                                                                                                                             |


## ADR coordination

- **W4 parity matrix** and **ADR-A02 Epic C MCP-vs-Rust-CLI delta**: both produce capability tables. One canonical doc should own the MCP/CLI matrix; the other links. Implementer should resolve via `coord-mcp-rvf-doc-canonical` in the ecosystem plan before W4.
- **W0 semver**: coordinate with ADR-A02 if rvlite (`getStorageBackend()` label change) ships in the same npm release train.

## Related Decisions

- **ADR-029**: RVF canonical format — this ADR aligns the MCP/CLI interface with ADR-029's persistence contracts
- **ADR-001**: Core architecture — MCP tools are the agent-facing surface of the architecture described here

## Related Plans

- [CLI MCP nomenclature todos](../plans/cli_mcp_nomenclature_todos_e2d8271a.plan.md) — per-todo implementation details for W0-W2
- [CLI MCP ecosystem todos](../plans/cli_mcp_ecosystem_research_todos_beb308b1.plan.md) — per-todo details for E1-E5
- [Research streams execution order](../plans/research_streams_execution_order_d0f4d7b3.plan.md) — cross-stream wave sequencing

## Revision History


| Version | Date       | Author    | Changes                              |
| ------- | ---------- | --------- | ------------------------------------ |
| 0.1     | 2026-04-10 | clawdette | Initial proposal from research audit |
