---
name: CLI MCP ecosystem research todos
overview: Implementation, testing, and documentation backlog derived from [CLI_MCP_ECOSYSTEM_RESEARCH.md](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md), framed with structured completeness criteria and sequential-thinking dependencies (including prerequisites from the nomenclature backlog).
todos:
  - id: w0-nomenclature-prereq
    content: "W0: Complete or coordinate nomenclature P0 (rvf_create dimensions, rvf_delete ids, workers_dispatch) before CLI rvf delete"
    status: pending
  - id: e1-cli-rvf-delete
    content: "E1: Add CLI `rvf delete` (cli.js + rvf-wrapper) aligned with MCP rvf_delete semantics"
    status: pending
  - id: e1-mcp-rvf-export-or-docs
    content: "E1: Add MCP rvf_export OR document explicit export workaround (CLI/script)"
    status: pending
  - id: e2-workers-cancel-cleanup-mcp
    content: "E2: Add MCP tools for workers cancel + cleanup mirroring CLI"
    status: pending
  - id: e4-shared-validate-rvf-path
    content: "E4: Extract validateRvfPath to shared module; optional CLI --allow-outside-cwd; document MCP vs CLI path policy"
    status: pending
  - id: test-mcp-smoke
    content: "§6.1: MCP smoke test — ListTools + CallTool per critical group (rvf_create temp, hooks_stats)"
    status: pending
  - id: test-rvf-path-traversal
    content: "§6.3: Test MCP validateRvfPath rejects traversal; optional CLI rvf mirror"
    status: pending
  - id: e3-single-manifest
    content: "E3: Single tool manifest (toolId, schema, cliEquivalent, handler); generate tools/list + mcp tools output"
    status: pending
  - id: e3-split-lazy-mcp-server
    content: "§7.1: Split mcp-server by domain + lazy require per group"
    status: pending
  - id: e3-lazy-intelligence-engine
    content: "§7.5: Defer IntelligenceEngine construction until first hooks tool that needs it"
    status: pending
  - id: test-parity-manifest-snapshot
    content: "§6.2: Parity checklist test from manifest (tool count, new tools presence)"
    status: pending
  - id: docs-parity-surface-matrix
    content: "W5: README/doc — golden paths §5; CLI-only hooks/native/brain; hooks_algorithms_list vs learning-config"
    status: pending
  - id: e5-rvlite-cli-or-mcp-only-doc
    content: "E5: Thin `ruvector rvlite` CLI OR document MCP-only rvlite queries"
    status: pending
  - id: docs-mcp-server-env
    content: "§7.4: Document MCP_SERVER=1 vs CLI worker behavior in README"
    status: pending
  - id: product-vector-db-mcp
    content: "Product decision: dedicated MCP tools for vector DB insert/search — defer unless approved"
    status: pending
  - id: coord-mcp-rvf-doc-canonical
    content: Coordinate canonical doc for MCP vs Rust rvf CLI + ruvector parity — one owner with cross-links (avoid duplicating persistence p2-c2 + nomenclature readme in full)
    status: pending
  - id: coord-rvlite-doc-persistence
    content: E5 rvlite MCP-only/thin CLI wording must align with RVF persistence Epic A (rvlite honesty) — same release train or explicit cross-links
    status: pending
  - id: release-semver-w0-w1
    content: "Cross-cutting: coordinated CHANGELOG entry for W0 P0 breaking schema fixes + W1 E1/E2 new tools (workers_cancel, CLI rvf delete, rvf_export); one npm version bump per wave, not per todo"
    status: pending
  - id: security-review-e2-e4
    content: "Security: short threat-model note for E2 workers MCP (shell exec surface) + E4 validateRvfPath extraction (path traversal); review before shipping, not after"
    status: pending
  - id: e3-perf-acceptance
    content: "E3 acceptance: measure cold list_tools latency + startup RSS before/after split/lazy refactor; target <200ms list_tools, <50MB RSS idle; document baseline first"
    status: pending
isProject: false
---

# Todos derived from CLI / MCP ecosystem research

**Source:** [CLI_MCP_ECOSYSTEM_RESEARCH.md](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)  
**Scope:** [`npm/packages/ruvector`](npm/packages/ruvector) only (`bin/mcp-server.js`, `bin/cli.js`, tests, README).

## Framing (how this backlog was built)

Applied as a **completeness and prioritization gate** (not a full design doc):

- **Context:** 95 MCP tools vs large Commander CLI; neither surface is a subset of the other (§1).
- **Success criteria:** Every **§8 gap** maps to a todo or an explicit **product decision**; **§9 epics** have shippable slices; **§6** adds measurable test coverage; **§3/§4** “document vs implement” gaps are labeled.
- **Approaches:**
  - **A (recommended):** Ship **parity for dangerous asymmetries** first (delete/export, workers lifecycle), then **path security**, then **manifest/drift**, then **optional rvlite CLI**.
  - **B:** **Manifest (E3) first** — reduces drift early but churns while parity gaps and [nomenclature](../analysis/ruvector/CLI_MCP_NOMENCLATURE_RESEARCH.md) contract fixes are still moving.
  - **C:** **Docs-only** for all gaps — fast but leaves agents without cancel/cleanup/export on MCP.

**Recommendation:** **A**, with **nomenclature P0** completed or coordinated before **CLI `rvf delete`** so ID/types match MCP ([`cli_mcp_nomenclature_todos_e2d8271a.plan.md`](./cli_mcp_nomenclature_todos_e2d8271a.plan.md)).

## Sequential thinking (dependency summary)

1. **E1 RVF parity** — CLI `rvf delete` should align with **fixed** `rvf_delete` MCP contract (string IDs, dimensions) — **after or with** nomenclature P0.
2. **MCP `rvf_export`** (or documented workaround) — independent of delete but same **mcp-server.js** touch surface; pair in one milestone when possible.
3. **E2 workers MCP** — Add `workers_cancel` / `workers_cleanup` (names TBD) mirroring CLI; **after** [`workers_dispatch` honest failure](./cli_mcp_nomenclature_todos_e2d8271a.plan.md) for consistent agent error handling.
4. **E4 path policy** — Shared `validateRvfPath` module + CLI escape hatch + docs; drives **§6.3** path tests.
5. **E3 manifest** — **After** E1/E2 change tool inventory/schemas; generates **§6.2** parity checklist test.
6. **E5 rvlite thin CLI** — Lowest; alternative is **README “MCP-only”** for `rvlite_*`.
7. **§7 optimizations** — Split/lazy `mcp-server`, lazy `IntelligenceEngine` — bundle with E3 or follow as refactor milestone.
8. **Product fork:** **Vector DB `insert`/`search` on MCP** — decision only unless product approves.

## Wave model (aligned with research §9)

| Wave | Epic | Tasks (from research) |
|------|------|------------------------|
| **W0** | Prerequisite | Complete or coordinate [nomenclature P0](./cli_mcp_nomenclature_todos_e2d8271a.plan.md): `rvf_create` dimensions, `rvf_delete` types, `workers_dispatch` failure JSON |
| **W1** | **E1** | **CLI `rvf delete`** ([§8](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) — `bin/cli.js` + [`src/core/rvf-wrapper.ts`](npm/packages/ruvector/src/core/rvf-wrapper.ts) (or dist equivalent); mirror MCP semantics |
| **W1** | **E1** | **`rvf_export` MCP tool** **or** explicit **docs workaround** (export via CLI / script) — [`mcp-server.js`](npm/packages/ruvector/bin/mcp-server.js) |
| **W2** | **E2** | **`workers_cancel` / `workers_cleanup` MCP** — shell/exec parity with CLI ([§4–5](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **W3** | **E4** | **Path policy:** export shared validator module; document MCP strict vs CLI user paths; optional CLI `--allow-outside-cwd` ([§7.3](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md), §4 matrix) |
| **W3** | **Tests** | **RVF path policy test** — MCP rejects traversal; optional CLI mirror ([§6.3](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **W4** | **E3** | **Single tool manifest** (JSON/TS): `toolId`, schema, `cliEquivalent`, `handler` — generate `tools/list` + `ruvector mcp tools` ([§7.2](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **W4** | **E3 + §7** | **Split `mcp-server.js` by domain**; **lazy `require` per group**; **defer `IntelligenceEngine`** until first consuming hooks tool ([§7.1](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md), §7.5) |
| **W4** | **Tests** | **Parity checklist test** from manifest — tool count / presence of new tools ([§6.2](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **W2–W3** | **Tests** | **MCP smoke:** `ListTools` + one `CallTool` per critical group (e.g. temp file `rvf_create`, `hooks_stats`) — stdio mock or `ruvector mcp test` ([§6.1](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **W5** | **Docs** | **Unified parity doc / README:** golden paths §5; CLI-only hooks (`session-start`, …), `native run`, `hooks_algorithms_list` vs `learning-config --list`, brain MCP-only ([§3–4](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **W5** | **E5** | **`ruvector rvlite` thin CLI** **or** document **MCP-only** `rvlite_sql` / cypher / sparql ([§8](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **Backlog** | Product | **Vector DB on MCP** — `insert` / `search` as dedicated tools if desired ([§8](../analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md)) |
| **W5** | **§7.4** | Document **`MCP_SERVER=1`** vs CLI worker behavior in README |

## Coverage audit (research section → work)

| Section | Captured in |
|---------|-------------|
| §1 executive gaps | E1, E2, E5, backlog vector MCP, W5 docs |
| §2 inventory (95 tools) | E3 manifest + §6.2 snapshot |
| §3 CLI tree / CLI-only | W5 docs |
| §4 parity matrix | W0–W5 tasks + path row → E4 |
| §5 golden paths | W5 docs / examples |
| §6 test map + recommendations | MCP smoke, path test, parity checklist |
| §7 optimization | E3 milestone + optional perf follow-up |
| §8 gap list | All rows → todos below |
| §9 epic order | W0→W1→W2→W3→W4→W5 |

## Key files

- [`npm/packages/ruvector/bin/mcp-server.js`](npm/packages/ruvector/bin/mcp-server.js)
- [`npm/packages/ruvector/bin/cli.js`](npm/packages/ruvector/bin/cli.js)
- [`npm/packages/ruvector/src/core/rvf-wrapper.ts`](npm/packages/ruvector/src/core/rvf-wrapper.ts)
- [`npm/packages/ruvector/test/cli-commands.js`](npm/packages/ruvector/test/cli-commands.js)
- [`npm/packages/ruvector/test/integration.js`](npm/packages/ruvector/test/integration.js)

## Security review (E2 + E4) — before shipping

| Surface | Threat | Review scope |
|---------|--------|-------------|
| **E2 `workers_cancel` / `workers_cleanup`** | Shell exec — agent-supplied worker IDs reach `child_process`; cancellation of wrong PID | Validate worker ID format (alphanumeric + dash); confirm kill targets only known child PIDs; no user-controlled shell interpolation |
| **E4 `validateRvfPath` extraction** | Path traversal — shared module must not weaken MCP's strict policy when CLI adopts it with `--allow-outside-cwd` | Unit tests for `../`, symlinks, null bytes; CLI escape hatch is opt-in flag only; MCP path remains cwd-locked |

Short threat-model note (README or inline comment) is sufficient; no formal ADR required unless the review surfaces a design change.

## E3 performance acceptance criteria

Measure **before** refactoring (baseline), then **after** split/lazy work:

| Metric | Baseline (capture first) | Target |
|--------|--------------------------|--------|
| Cold `list_tools` latency (stdio round-trip) | TBD | <200ms |
| Startup RSS (idle, no tool calls) | TBD | <50MB |
| First `hooks_stats` call (triggers IntelligenceEngine) | TBD | <500ms |

Capture baseline with: `time echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node bin/mcp-server.js` and `/usr/bin/time -l` (macOS) or `command time -v` (Linux) for RSS.

## Cross-references

- Nomenclature contract fixes: [`cli_mcp_nomenclature_todos_e2d8271a.plan.md`](./cli_mcp_nomenclature_todos_e2d8271a.plan.md)
- RVF persistence implementation todos: [`rvf_persistence_research_todos_45f08914.plan.md`](./rvf_persistence_research_todos_45f08914.plan.md)
- Cross-stream ordering: [`research_streams_execution_order_d0f4d7b3.plan.md`](./research_streams_execution_order_d0f4d7b3.plan.md)
- Related persistence context: [RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md](../analysis/rvf/RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md)
- Semver/changelog coordination: [execution-order `cross-cutting-semver-changelog`](./research_streams_execution_order_d0f4d7b3.plan.md)

## Cross-plan audit (structured review + sequential thinking)

**Completeness gate:** Each sibling plan has a distinct primary scope; overlaps are **document owners** or **sequenced prerequisites**, not silent duplicate work.

**Sequential dependencies:**

| This plan todo | Depends on / coordinates with |
|----------------|------------------------------|
| `w0-nomenclature-prereq` | [Nomenclature P0](./cli_mcp_nomenclature_todos_e2d8271a.plan.md) (`rvf-create-dimensions`, `rvf-delete-ids`, `test-delete-string-ids`, `workers-dispatch-failure`) |
| `e1-cli-rvf-delete` | Same + `rvf-wrapper-review` so CLI and MCP share semantics |
| `e2-workers-cancel-cleanup-mcp` | Nomenclature `workers-dispatch-failure` first |
| `e4-shared-validate-rvf-path` | Complements nomenclature `readme-mcp-cli-table` (path vocabulary); implement **after** P0 stabilizes behavior |
| `e3-single-manifest` | Nomenclature `optional-schema-codegen` + `coord-optional-codegen-manifest` — **one** generator or manifest owns schemas |
| `e5-rvlite-cli-or-mcp-only-doc` | [Persistence Epic A](./rvf_persistence_research_todos_45f08914.plan.md) (`p0-a1`–`a4`) so public wording matches on-disk truth |
| `docs-parity-surface-matrix` | **Merge** Rust `rvf` CLI / MCP delta with persistence `p2-c2-mcp-cli-delta` (see `coord-mcp-rvf-doc-canonical`) |

**Non-duplication:**

| Topic | Owner plan | Other plans |
|-------|------------|-------------|
| MCP JSON schema / `workers_dispatch` errors | Nomenclature | Ecosystem consumes stable contracts |
| `validateRvfPath` extraction | Ecosystem E4 | Nomenclature documents `db_path`→`path` only |
| rvlite `saveToRvf` truth | Persistence P0 | Ecosystem E5 docs reference it |
| MCP smoke (`ListTools` / `CallTool`) | Ecosystem `test-mcp-smoke` | Nomenclature error-contract tests are **JSON shape** focused |
| Node RVF lifecycle (`@ruvector/rvf`) | Persistence `p1-t1-node-lifecycle` | Ecosystem smoke may use **MCP** `rvf_create` — complementary layers |
