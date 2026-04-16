# RuVector npm package — CLI / MCP nomenclature and ergonomics

**Scope:** [`npm/packages/ruvector`](../../../npm/packages/ruvector) — MCP tool `inputSchema` keys/types, handler argument use, [`src/core/rvf-wrapper`](../../../npm/packages/ruvector/src/core/rvf-wrapper.ts) / [`@ruvector/rvf`](../../../npm/packages/rvf), and Commander CLI naming. **Sources:** [`bin/mcp-server.js`](../../../npm/packages/ruvector/bin/mcp-server.js), [`bin/cli.js`](../../../npm/packages/ruvector/bin/cli.js), [`npm/packages/rvf/src/types.ts`](../../../npm/packages/rvf/src/types.ts), [`npm/packages/rvf/src/database.ts`](../../../npm/packages/rvf/src/database.ts). **Non-goals:** Rust `rvf-cli`; other packages’ MCP servers.

---

## 1. Executive summary — top naming/type mismatches

| # | Issue | Severity | Notes |
|---|--------|----------|--------|
| 1 | **`rvf_create` passes `dimension` into `createRvfStore`**, but `RvfOptions` and `RvfDatabase.create` expect **`dimensions`** | **High** | MCP schema and copy use `dimension`; CLI correctly uses `dimensions: opts.dimension`. Native mapping uses **`options.dimensions`** only (`mapOptionsToNative` in [`backend.ts`](../../../npm/packages/rvf/src/backend.ts)); passing **`dimension`** leaves **`dimensions` undefined** at runtime — misaligned with the TS contract and risky for create. |
| 2 | **`rvf_delete` schema types `ids` as `number[]`**, while **`RvfDatabase.delete(ids: string[])`** and **`RvfSearchResult.id`** are **string**-oriented | **High** | Strict MCP clients sending string IDs fail JSON Schema validation; numeric IDs may still coerce oddly vs u64 string IDs in the wild. |
| 3 | **`rvf_ingest.entries`** is `items: { type: 'object' }` with **no property schema** | Medium | Undocumented shape vs `RvfIngestEntry` (`id`, `vector`, optional metadata); runtime errors only clarify shape. |
| 4 | **`rvlite_*` MCP key `db_path`** → handler maps to **`{ path: validateRvfPath(args.db_path) }`** for `rvlite.Database` | Medium | Same meaning, different public name than rvlite’s `path` option; docs should state the mapping. |
| 5 | **`hooks_recall` uses `top_k`** (snake); CLI **`--top-k`** surfaces as Commander **`opts.topK`** (camel) | Low | Alias risk for authors who mirror CLI mental model (`topK`) into MCP JSON. |
| 6 | **`workers_status` uses `workerId`** (camelCase) amid mostly **snake_case** MCP keys | Low | Inconsistent convention; CLI positional is `[workerId]` — aligns with MCP camel choice. |
| 7 | **`workers_dispatch` catch branch returns `success: true`** with a note | **High (ergonomics)** | Agents cannot distinguish failure without parsing stderr; no `hint` or stable error code — poor retry signal. |
| 8 | **`rvf_derive`**: MCP **`parent_path` / `child_path`** vs CLI **`<parent> <child>`** positionals | Low | Semantic match; key names differ from `rvf create <path>` single `path`. |
| 9 | **`RvfOptions`** uses **`efConstruction`** (camel); MCP `rvf_create` does not expose `ef_construction` / `m` / `compression` (CLI `rvf create` also only `-d`/`-m`) | Medium | Capability/schema drift between surfaces and full library options. |
| 10 | Most errors are **`{ success: false, error: e.message }`** without **`expectedKeys` / field paths** | Medium | Optional deps add **`hint`** (`npm install …`); validation failures rarely teach correct MCP key names. |

---

## 2. Per-domain tables

Columns: **MCP key** → **MCP schema type** → **Handler mapping** → **CLI equivalent** → **Library / runtime key** → **Match** → **Error notes**.

### 2.1 RVF

| MCP tool | MCP_key | MCP_schema_type | handler_mapping | CLI_flag_or_arg | library_key | match_status | error_quality_notes |
|----------|---------|-----------------|-----------------|-----------------|-------------|--------------|---------------------|
| `rvf_create` | `path` | string | `validateRvfPath` → `createRvfStore(safePath, opts)` | `<path>` positional | `path` (1st arg to `create`) | **exact** | Missing RVF: `hint` to install `@ruvector/rvf` |
| `rvf_create` | `dimension` | number | `{ dimension: args.dimension, metric }` passed to wrapper | `-d, --dimension <n>` → **`dimensions: opts.dimension`** | **`dimensions`** in `RvfOptions` | **key mismatch** | `e.message` from library may not say “use dimensions” |
| `rvf_create` | `metric` | string | same | `-m, --metric` | `metric?` | **exact** | — |
| `rvf_open` | `path` | string | `openRvfStore` | (no direct CLI) | `path` | **exact** | Generic `e.message` |
| `rvf_ingest` | `path` | string | `openRvfStore` + `rvfIngest(store, args.entries)` | `<path>`, `-i, --input` file | `ingestBatch(entries)` | **path exact**; **entries loose schema** | Generic |
| `rvf_query` | `path` | string | `openRvfStore` | `<path>` | — | **exact** | Generic |
| `rvf_query` | `vector` | number[] | `rvfQuery(store, args.vector, args.k \|\| 10)` | `-v, --vector` CSV → `Number` | `query(vector, k, …)` | **exact** | No `filter` / `efSearch` in MCP (library supports `RvfQueryOptions`) |
| `rvf_query` | `k` | number | 3rd arg to `rvfQuery` | `-k, --k` | `k` | **exact** | — |
| `rvf_delete` | `path` | string | `rvfDelete(store, args.ids)` | **no CLI** | **`delete(ids: string[])`** | **`ids` type mismatch** (schema `number[]`) | Generic |
| `rvf_status` / `compact` / `segments` | `path` | string | `validateRvfPath` + wrapper | `<path>` | `path` | **exact** | Generic |
| `rvf_derive` | `parent_path`, `child_path` | string | `rvfDerive(store, child_path)` | `<parent> <child>` | `derive(childPath)` | **alias** (names differ, semantics match) | Generic |
| `rvf_examples` | `filter` | string | in-handler filter | `rvf examples` (no filter flag in snippet) | — | **partial** | N/A |

### 2.2 `rvf-wrapper` / `@ruvector/rvf` canonical shapes

| Concept | Canonical field / API | Notes |
|---------|----------------------|--------|
| Create options | `RvfOptions`: **`dimensions`**, `metric?`, `profile?`, `compression?`, `signing?`, `m?`, `efConstruction?` | MCP handler should pass **`dimensions`** (and optionally map `ef_construction` → `efConstruction`). |
| Ingest entry | `RvfIngestEntry`: **`id: string`**, **`vector`**, optional payload/metadata per types | MCP should document or constrain `entries[]` accordingly. |
| Delete | `RvfDatabase.delete(ids: string[])` | Align MCP `ids` with **`string` or `oneOf` string|number** if clients send both. |
| Query | `query(vector, k, options?)` | MCP omits `options` object; consider `ef_search`, `filter` later. |

### 2.3 Workers (agentic-flow bridge)

| MCP tool | MCP_key | MCP_schema_type | handler_mapping | CLI | match_status | error_quality_notes |
|----------|---------|-----------------|-----------------|-----|--------------|---------------------|
| `workers_dispatch` | `prompt` | string | `execSync('… workers dispatch "${prompt}"')` | `<prompt...>` | **exact** | **On failure: still `success: true`** — misleading |
| `workers_status` | `workerId` | string | optional shell arg | `[workerId]` | **exact** (camelCase) | `success: false` + `message` |
| `workers_results` | `json` | boolean | `--json` flag | `--json` | **exact** | — |
| `workers_create` | `name`, `preset`, `triggers` | string (+ optional) | shell | `<name>`, `--preset`, `--triggers` | **exact** | Generic on throw |
| `workers_run` | `name`, `path` | string | shell | `<name>`, `--path` | **exact** | Generic |
| `workers_init_config` | `force` | boolean | `--force` | `--force` | **exact** | — |
| `workers_load_config` | `file` | string | `--file` | `--file` | **exact** | — |
| Stateless listing tools | — | empty object | `execSync` | same subcommands | **exact** | — |

### 2.4 Hooks (aggregate + exceptions)

**Pattern (majority):** MCP tool name `hooks_snake_case` ↔ CLI `hooks kebab-case`; handler either calls **`intel.*(args...)`** with same property names as `inputSchema`, or builds **`npx ruvector hooks <kebab> …`** from `args`.

| Area | match_status | Exceptions / notes |
|------|--------------|-------------------|
| `hooks_route`, `hooks_remember`, `hooks_recall`, trajectories, co-edit, errors | **exact** key pass-through to `intel` | `hooks_recall.top_k` vs CLI `opts.topK` naming style |
| `hooks_init` | **partial vs CLI** | MCP: `pretrain`, `build_agents`, `force` → CLI `--pretrain`, `--build-agents`, `--force`. CLI also has **`--minimal` / `--fast`** etc. — **not** on MCP schema. |
| `hooks_pretrain` | **exact** | `depth`, `skip_git`, `verbose` → `--depth`, `--skip-git`, `--verbose` |
| `hooks_build_agents` | **exact** | `focus`, `include_prompts` → `--focus`, `--include-prompts` |
| Shell-delegated analysis tools | **exact** | e.g. `hooks_ast_complexity`: `files`, `threshold` → positional files + `--threshold`; `hooks_diff_similar`: `top_k`, `commits` → `-k`, `--commits`; `hooks_git_churn`: `days`, `top` → `--days`, `--top` |
| `hooks_route_enhanced` | **exact** | `task`, `file` → quoted task + `--file` |

**Undocumented / loose schemas:** Several tools use `type: 'object'` without `properties` for nested blobs (e.g. learning engine payloads where applicable); treat as **undocumented** until tightened.

### 2.5 rvlite

| MCP tool | MCP_key | Handler → runtime | match_status | error_quality_notes |
|----------|---------|-------------------|--------------|---------------------|
| `rvlite_sql` / `cypher` / `sparql` | `query` | `sanitizeShellArg` then `db.sql/cypher/sparql` | **exact** | Missing package: **`hint`** `npm install rvlite` |
| same | `db_path` | `{ path: validateRvfPath(args.db_path) }` if set | **rename** (`db_path` → `path`) | Path errors via `e.message` |

### 2.6 Brain (`@ruvector/pi-brain`)

| MCP tool | Keys (schema) | Handler | CLI | match_status | error_quality_notes |
|----------|---------------|---------|-----|--------------|---------------------|
| `brain_search` | `query`, `limit`, `category` | `client.search({ … })` | `search <query>`, `-l`, `-c` | **exact** | MODULE_NOT_FOUND: **`hint`** install |
| `brain_share` | `title`, `content`, `category`, `tags[]` | `client.share({ … })` | `share <title>`, `-c`, `-t`, `--content` | **exact** (CLI tags CSV → array) | — |
| `brain_get` / `vote` / `delete` | `id`, `direction` | client methods | positionals | **exact** | — |
| `brain_list` | `category`, `limit` | `client.list(…)` | `-c`, `-l` | **exact** | — |
| `brain_partition` | `domain`, `min_cluster_size` | `client.partition({ domain, min_cluster_size })` | (if exposed similarly) | **exact** | — |
| `brain_transfer` | `source`, `target` | `client.transfer(args.source, args.target)` | — | **exact** | — |
| `brain_sync` | `direction` | `client.sync({ direction })` | — | **exact** | — |

### 2.7 Edge & identity

| Tool | MCP keys | Handler pattern | CLI parity | Notes |
|------|----------|-----------------|------------|--------|
| `edge_*` | `contribution`, `key` (optional) | lazy `require` of edge client | See CLI `edge` if present | Mostly env-driven (`PI`); key naming consistent. |
| `identity_*` | `key` optional | identity helpers | CLI counterparts | Low surface area; aligned. |

### 2.8 Decompile

| Tool | MCP keys | Notes |
|------|----------|--------|
| `decompile_package` | `package`, `version?`, `min_confidence` | snake_case matches handler expectations |
| `decompile_file` | `path`, `min_confidence` | `path` shared word with RVF but different domain |
| `decompile_url` | `url`, `min_confidence` | — |
| `decompile_search` | `query`, `package?`, `version?`, `path?` | — |
| `decompile_diff` | `package`, `version_a`, `version_b` | snake_case version keys |
| `decompile_witness` | `witness_path`, `source_path?` | — |

---

## 3. Cross-cutting conventions

| Topic | Convention | Implication |
|-------|------------|-------------|
| MCP vs CLI token style | **`snake_case`** args in JSON vs **kebab-case** flags | Document a **stable mapping table** for authors (e.g. `build_agents` → `--build-agents`). |
| Path naming | RVF uses **`path`**; derive uses **`parent_path` / `child_path`**; rvlite uses **`db_path`** | Prefer one vocabulary per subsystem or document aliases explicitly. |
| CamelCase islands | `workerId`, `hooks_route_enhanced` N/A | Accept as-is or standardize to snake_case in a breaking MCP revision. |
| Boolean CLI flags | `workers_init_config.force` | Mirrors `--force` cleanly. |

---

## 4. Ergonomic recommendations

1. **Fix `rvf_create` handler** to pass **`dimensions: args.dimension`** (or accept **`dimensions`** in schema and alias `dimension` for backward compatibility).
2. **Change `rvf_delete.ids`** to **`array` of `string`**, or JSON Schema **`oneOf`** string/number if both must be supported; coerce to string in handler before `store.delete`.
3. **Tighten `rvf_ingest.entries`** with `items` properties matching `RvfIngestEntry` (at least `id`, `vector`, optional metadata).
4. **`workers_dispatch`:** on `execSync` failure return **`success: false`**, include **`stderr` / exit code**, and a **`hint`** (e.g. check `workers_status`, install `agentic-flow`).
5. **Structured errors:** extend failure JSON with **`code`**, **`field`**, **`expectedKeys`** where validation is under server control (custom validation before delegate).
6. **README / doc table:** “MCP argument → CLI equivalent” for RVF, workers, hooks_init, brain_search/share (high-traffic tools).

---

## 5. Error sampling (representative)

| Scenario | Typical payload | Teaches correct key/type? |
|----------|-----------------|---------------------------|
| Missing `@ruvector/rvf` on `rvf_create` | `success: false`, `error`, **`hint`: npm install** | **Actionable** (install), not schema-level |
| Missing `rvlite` | `hint`: `npm install rvlite` | **Actionable** |
| Missing `@ruvector/pi-brain` | `hint`: install package | **Actionable** |
| Generic RVF throw | `error: e.message` only | **Depends on library message** — often not MCP-key-specific |
| `workers_dispatch` subprocess failure | **`success: true`** + note | **Poor** — no retry signal |
| MCP SDK schema validation (client-side) | implementation-defined | Wrong type on `rvf_delete.ids` may **block call** before server explains string IDs |

---

## 6. Suggested epics (implementation follow-up)

1. **RVF MCP alignment:** `dimensions` + optional `ef_construction` → `efConstruction`, `m`, `compression` parity with `RvfOptions` where intended.
2. **`rvf_delete` contract:** schema + handler + integration test with string u64 IDs from ingest results.
3. **Workers dispatch honesty:** structured failure object; document agent retry pattern.
4. **Error contract tests:** assert JSON responses for N failure modes include `hint` or `field` where applicable.
5. **Optional:** generate MCP `inputSchema` fragments from **`@ruvector/rvf` TypeScript types** to prevent drift.

---

## 7. Appendix — tool count reference

See [CLI_MCP_ECOSYSTEM_RESEARCH.md](./CLI_MCP_ECOSYSTEM_RESEARCH.md) §2 for the full **95-tool** inventory by group; this document focuses on **naming, types, and error ergonomics** rather than capability parity.
