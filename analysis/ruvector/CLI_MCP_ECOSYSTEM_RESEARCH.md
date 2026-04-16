# RuVector npm package — CLI vs MCP ecosystem research

**Scope:** [`npm/packages/ruvector`](../../../npm/packages/ruvector) only (not separate `pi-brain` MCP transports). **Artifacts:** [`bin/mcp-server.js`](../../../npm/packages/ruvector/bin/mcp-server.js), [`bin/cli.js`](../../../npm/packages/ruvector/bin/cli.js). Inventory taken from source grep; counts exclude MCP **resource** display names (`Intelligence Stats`, `Learned Patterns`).

---

## 1. Executive summary

- **MCP** exposes **95 tools** for agents (hooks, workers, RVF, rvlite queries, brain, edge, identity, decompile) plus stdio transport and optional `IntelligenceEngine`.
- **CLI** is a large **Commander** tree: core **vector DB** commands (`create`, `insert`, `search`, …), **hooks** subtree (naming uses **kebab-case** vs MCP **snake_case**), **workers** (delegates to `agentic-flow`), **rvf** subcommands, **native** workers, **embed**, **decompile**, **mcp** meta-commands (`start`, `info`, `tools`, `test`), attention/GNN/doctor/setup, etc.
- **Neither surface is a subset of the other.** Notable gaps: **RVF delete** (MCP yes / CLI no), **RVF export** (CLI yes / MCP no), **workers cleanup & cancel** (CLI yes / MCP no), **rvlite SQL/Cypher/SPARQL** (MCP only), **vector insert/search** (CLI first-class / MCP not mirrored as dedicated tools).

---

## 2. MCP tool inventory (95)

| Group | Tools |
|-------|--------|
| Meta | `ruvector` |
| Hooks (51) | `hooks_stats`, `hooks_route`, `hooks_remember`, `hooks_recall`, `hooks_init`, `hooks_pretrain`, `hooks_build_agents`, `hooks_verify`, `hooks_doctor`, `hooks_export`, `hooks_capabilities`, `hooks_import`, `hooks_swarm_recommend`, `hooks_suggest_context`, `hooks_trajectory_begin`, `hooks_trajectory_step`, `hooks_trajectory_end`, `hooks_coedit_record`, `hooks_coedit_suggest`, `hooks_error_record`, `hooks_error_suggest`, `hooks_force_learn`, `hooks_ast_analyze`, `hooks_ast_complexity`, `hooks_diff_analyze`, `hooks_diff_classify`, `hooks_diff_similar`, `hooks_coverage_route`, `hooks_coverage_suggest`, `hooks_graph_mincut`, `hooks_graph_cluster`, `hooks_security_scan`, `hooks_rag_context`, `hooks_git_churn`, `hooks_route_enhanced`, `hooks_attention_info`, `hooks_gnn_info`, `hooks_learning_config`, `hooks_learning_stats`, `hooks_learning_update`, `hooks_learn`, `hooks_algorithms_list`, `hooks_compress`, `hooks_compress_stats`, `hooks_compress_store`, `hooks_compress_get`, `hooks_batch_learn`, `hooks_subscribe_snapshot`, `hooks_watch_status` |
| Workers (11) | `workers_dispatch`, `workers_status`, `workers_results`, `workers_triggers`, `workers_stats`, `workers_presets`, `workers_phases`, `workers_create`, `workers_run`, `workers_custom`, `workers_init_config`, `workers_load_config` |
| RVF (10) | `rvf_create`, `rvf_open`, `rvf_ingest`, `rvf_query`, `rvf_delete`, `rvf_status`, `rvf_compact`, `rvf_derive`, `rvf_segments`, `rvf_examples` |
| Rvlite (3) | `rvlite_sql`, `rvlite_cypher`, `rvlite_sparql` |
| Brain (12) | `brain_search`, `brain_share`, `brain_get`, `brain_vote`, `brain_list`, `brain_delete`, `brain_status`, `brain_drift`, `brain_partition`, `brain_transfer`, `brain_sync` |
| Edge (4) | `edge_status`, `edge_join`, `edge_balance`, `edge_tasks` |
| Identity (2) | `identity_generate`, `identity_show` |
| Decompile (7) | `decompile_package`, `decompile_file`, `decompile_url`, `decompile_search`, `decompile_diff`, `decompile_witness` |

---

## 3. CLI command tree (high level)

**Top-level** (non-exhaustive; nested under `gnn`, `attention`, `embed`, `hooks`, `workers`, `native`, `rvf`, `mcp`, `decompile`, etc.):

- **Vector DB:** `create`, `insert`, `search`, `stats`, `benchmark`, `export`, `import`, `info`, `install`, `doctor`, `setup`, …
- **GNN / attention / graph / router / server / cluster:** dedicated subtrees (`gnn`, `attention`, `graph`, `router`, `server`, `cluster`, …).
- **Embed:** `embed text|adaptive|benchmark|optimized|neural|demo`.
- **Hooks:** `hooks <subcommand>` — maps loosely to MCP `hooks_*` (e.g. `hooks stats` ↔ `hooks_stats`, `hooks route` ↔ `hooks_route`). Extra **CLI-only** ergonomics: `session-start`, `session-end`, `pre-edit`, `post-edit`, `pre-command`, `post-command`, `async-agent`, `lsp-diagnostic`, `track-notification`, `pre-compact`, etc. **Algorithms listing:** MCP `hooks_algorithms_list` vs CLI `hooks learning-config --list` (no dedicated `hooks algorithms-list`).
- **Workers:** `workers dispatch|status|results|triggers|stats|cleanup|cancel|presets|phases|create|run|custom|init-config|load-config` — **cleanup** and **cancel** have **no** MCP tool.
- **Native:** `native run` — **CLI-only** (ONNX/native workers).
- **RVF:** `create`, `ingest`, `query`, `status`, `segments`, `derive`, `compact`, `export`, `examples`, `download` — **no** `delete` subcommand; **export** has **no** MCP `rvf_export`.
- **MCP meta:** `mcp start|info|tools|test`.
- **Decompile:** `decompile [target]` with flags — full CLI flow; MCP splits into `decompile_*` tools.

---

## 4. Parity matrix (capability × MCP × CLI × notes)

| Capability | MCP | CLI | Shared / notes |
|------------|-----|-----|----------------|
| RVF create | `rvf_create` | `rvf create` | [`dist/core/rvf-wrapper.js`](../../../npm/packages/ruvector/dist/core/rvf-wrapper.js); MCP uses **`validateRvfPath`** |
| RVF open / probe | `rvf_open` | (no direct; use ingest/query) | MCP-only convenience |
| RVF ingest | `rvf_ingest` (entries in args) | `rvf ingest --input file` | JSON file on CLI |
| RVF query | `rvf_query` | `rvf query -v` | |
| RVF delete | **`rvf_delete`** | **missing** | Gap: agents can delete; CLI users cannot without scripting |
| RVF status | `rvf_status` | `rvf status` | |
| RVF compact | `rvf_compact` | `rvf compact` | |
| RVF derive | `rvf_derive` | `rvf derive` | |
| RVF segments | `rvf_segments` | `rvf segments` | |
| RVF examples | `rvf_examples` | `rvf examples`, `rvf download` | CLI richer for downloads |
| RVF export JSON | **missing** | **`rvf export`** | Gap: no MCP export tool |
| RVF path security | **Strict** (cwd + blocked prefixes) | **User paths** | Document or align policy |
| Workers dispatch | `workers_dispatch` | `workers dispatch` | Both delegate to agentic-flow pattern |
| Workers cancel / cleanup | **missing** | **`workers cancel`**, **`workers cleanup`** | Lifecycle gap on MCP |
| Rvlite queries | **`rvlite_sql`**, **cypher**, **sparql** | **missing** | Scripting gap |
| Brain / edge / identity | `brain_*`, `edge_*`, `identity_*` | Documented under `mcp tools` | No `ruvector brain` CLI |
| Decompile | Granular tools | `ruvector decompile` | Different UX; same domain |
| Vector DB insert/search | **no dedicated tools** | **`insert`**, **`search`**, … | Asymmetric by design? document |
| Hooks algorithms list | `hooks_algorithms_list` | `hooks learning-config --list` | Different entrypoint |
| Native deep workers | — | **`native run`** | CLI-only |

---

## 5. Golden paths

### A. RVF lifecycle

| Step | MCP | CLI |
|------|-----|-----|
| Create | `rvf_create` | `rvf create` |
| Ingest | `rvf_ingest` | `rvf ingest` |
| Query | `rvf_query` | `rvf query` |
| Delete | `rvf_delete` | **—** |
| Compact | `rvf_compact` | `rvf compact` |
| Export snapshot | **—** | `rvf export` |

### B. Hooks (stats + route)

| Step | MCP | CLI |
|------|-----|-----|
| Stats | `hooks_stats` | `hooks stats` |
| Route | `hooks_route` | `hooks route` |

Both paths exist; naming differs (`_` vs space).

### C. Workers

| Step | MCP | CLI |
|------|-----|-----|
| Dispatch | `workers_dispatch` | `workers dispatch` |
| Status | `workers_status` | `workers status` |
| Cancel | **—** | `workers cancel` |
| Cleanup | **—** | `workers cleanup` |

---

## 6. Test coverage map

| Suite | Path | Covers MCP? | Notes |
|-------|------|-------------|--------|
| `cli-commands.js` | [`npm/packages/ruvector/test/cli-commands.js`](../../../npm/packages/ruvector/test/cli-commands.js) | **No** (CLI only) | `mcp --help`, `mcp info`, `rvf --help`, `rvf examples`, `hooks --help`, `hooks stats/route/remember/recall`, `workers --help` |
| `integration.js` | [`npm/packages/ruvector/test/integration.js`](../../../npm/packages/ruvector/test/integration.js) | **No** | Module load, `VectorDB`, types |

**Recommended additions:**

1. **MCP smoke:** spawn `node bin/mcp-server.js` with mocked stdio or `ruvector mcp test` if it exercises `ListTools` + one `CallTool` per critical group (rvf_create on temp file, hooks_stats).
2. **Parity checklist test:** generated from manifest (future Epic E3) asserting MCP tool count and presence of `rvf_delete` / `workers_cleanup` when implemented.
3. **RVF path policy test:** MCP `validateRvfPath` rejects traversal; optional mirror for CLI `rvf` commands.

---

## 7. Optimization and maintainability

1. **Monolithic `mcp-server.js`** (~3.7k+ lines): split handlers by domain; lazy `require` per group to reduce cold start.
2. **Single manifest** (JSON/TS): `toolId`, JSON schema, `cliEquivalent`, `handler` — generate MCP `tools/list` and `ruvector mcp tools` output to prevent drift.
3. **Shared RVF validation:** export `validateRvfPath` to a small shared module; use in CLI `rvf` with `--allow-outside-cwd` escape hatch if needed.
4. **Env flags:** `MCP_SERVER=1` in MCP vs CLI comment disabling parallel workers — document interaction in README.
5. **IntelligenceEngine:** defer construction until first hooks tool that needs full engine (measure `list_tools` latency).

---

## 8. Gap list (owners)

| Gap | Area |
|-----|------|
| `rvf delete` CLI | `bin/cli.js` + `rvf-wrapper` |
| `rvf_export` MCP or documented workaround | `mcp-server.js` |
| `workers_cancel` / `workers_cleanup` MCP | `mcp-server.js` |
| Rvlite CLI or documented “MCP-only” | `cli.js` / README |
| Vector DB tools on MCP (if desired) | product decision + `mcp-server.js` |
| Path policy alignment | security + both bins |
| Tool/help drift | manifest epic |

---

## 9. Epic prioritization (final)

| Order | Epic | Rationale |
|-------|------|-----------|
| **E1** | RVF parity (CLI delete + MCP export **or** explicit docs) | Removes dangerous asymmetry (delete/export) |
| **E2** | Workers parity on MCP (cancel/cleanup) | Agents need same lifecycle as humans |
| **E4** | Path policy (document or shared validator) | Security consistency |
| **E3** | Single manifest for tools + CLI tool list | Prevents future drift; enables snapshot tests |
| **E5** | Optional `ruvector rvlite` thin CLI | Lowest priority unless scripting demand |

---

## 10. References

- [`npm/packages/ruvector/bin/mcp-server.js`](../../../npm/packages/ruvector/bin/mcp-server.js) — `validateRvfPath`, tool registry, `switch` dispatch
- [`npm/packages/ruvector/bin/cli.js`](../../../npm/packages/ruvector/bin/cli.js) — Commander tree, `rvfCmd`, `workersCmd`, `hooksCmd`, `mcpCmd`
- [`npm/packages/ruvector/dist/core/rvf-wrapper.js`](../../../npm/packages/ruvector/dist/core/rvf-wrapper.js) — shared RVF I/O
- Related: [RVF persistence research](../rvf/RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md)
