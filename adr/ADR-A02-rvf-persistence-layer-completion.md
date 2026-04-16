ADR-A02: RVF Persistence Layer Completion — Truth, Gaps, and Migration Path

**Status**: Proposed
**Date**: 2026-04-10
**Authors**: clawdette (research + analysis), for review by ruv.io
**Deciders**: ruv.io, Architecture Review Board
**SDK**: Claude-Flow
**Extends**: ADR-029 (RVF as canonical binary format)

## Context

### The Split-Brain Persistence Problem

ADR-029 established RVF as the canonical binary format across all RuVector libraries. The research audit ([RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md](../analysis/rvf/RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md)) reveals that **only the Node N-API path** is fully wired to on-disk canonical `.rvf` files. Every other path is incomplete, misleading, or unimplemented:


| Surface                                                  | Canonical `.rvf` on disk         | Actual status                                                                                                                                   |
| -------------------------------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Rust** `crates/rvf/` (rvf-runtime, CLI, adapters)      | Yes                              | Substantial implementation + integration tests                                                                                                  |
| **Node** `npm/packages/rvf` → `@ruvector/rvf-node` N-API | Yes                              | `NodeBackend` delegates to N-API `RvfStore`                                                                                                     |
| **WASM** (`WasmBackend` in TS)                           | **No**                           | `open()` and `openReadonly()` explicitly throw; in-memory only                                                                                  |
| `**@ruvector/rvf-wasm`** npm                             | Partial                          | Buffer-based `rvf_store_open(buf, len)` — no file paths, no `writeRvf`/`readRvf` exports                                                        |
| **rvlite** `npm/packages/rvlite`                         | **Broken**                       | `saveToRvf`/`loadFromRvf` check for `writeRvf`/`readRvf` on wasm module — those symbols do not exist; always falls back to JSON "RVF1" envelope |
| **ruvector npm default**                                 | Only if `@ruvector/core` missing | Prefers `@ruvector/core` (REDB); RVF is fallback, not default                                                                                   |
| **ruvector-core** Rust                                   | **No RVF**                       | Storage remains REDB; ADR-029 migration table row not yet implemented                                                                           |
| **RuVocal** `ui/ruvocal`                                 | N/A                              | In-memory Mongo-shaped store persisted as JSON with `format: "rvf-database"` — not binary RVF; naming overlap is a documentation hazard         |


### What Misleads Users and Agents Today

1. **rvlite claims RVF persistence it cannot deliver** — `getStorageBackend()` returns `'rvf'` when the package is merely installed; `saveToRvf` always writes JSON, never binary RVF
2. **WASM README promises browser/edge RVF** — `WasmBackend` is explicitly in-memory for store lifecycle
3. **RuVocal naming** — `rvf.ts` and `format: "rvf-database"` in JSON stores are not the binary RVF from ADR-029
4. **Node string ID sidecar** — `NodeBackend` maps string IDs to numeric labels with a best-effort sidecar JSON; copying a `.rvf` file without its sidecar silently loses ID mappings

### Research Source


| Document                                                                                       | Scope                                                                                                                                    |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| [RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md](../analysis/rvf/RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md) | Full runtime x package matrix, rvlite contract audit, WASM vs Node limitations, ADR-029 gap table, peripheral modules, recommended epics |


## Decision

### Adopt a truth-first approach: fix misleading APIs before building new features

The persistence layer should not overstate its capabilities. Fix honesty and documentation issues first, then extend durability to WASM, then migrate ruvector-core.

### Epic A — rvlite honesty (P0)

**Packages:** `npm/packages/rvlite`, `npm/packages/rvf-wasm`


| Task                      | Change                                                                                                                            | Acceptance                                             |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Document default init     | `createRvLite` does not wire `rvf_store_`* into WASM constructor; document or fix so `getStorageBackend()` label matches behavior | Label matches actual wiring                            |
| README/JSDoc honesty      | Align with real `saveToRvf`/`loadFromRvf` behavior (JSON envelope, not binary RVF)                                                | No false expectation of ADR-029 file from default path |
| Dead branch cleanup       | Remove dead `writeRvf`/`readRvf` checks OR implement Node binary path via `rvf_store_export` + `fs`                               | No dead code implying binary RVF exists                |
| `getStorageBackend()` fix | Return value must not overstate RVF wiring                                                                                        | String matches actual data flow                        |
| On-disk format test       | Assert actual file format (magic bytes for binary OR JSON structure)                                                              | CI catches regression if format changes                |


**Files:** `[npm/packages/rvlite/src/index.ts](../../npm/packages/rvlite/src/index.ts)`, `[npm/packages/rvlite/README.md](../../npm/packages/rvlite/README.md)`, `[npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts](../../npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts)`.

### Epic E — Sidecar robustness + `@ruvector/rvf` tests

**Package:** `npm/packages/rvf`


| Task                              | Change                                                                                                                                         |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Document failure mode             | `.rvf` without sidecar `mappings.json` — what happens on reopen (silent ID loss vs error)                                                      |
| Sidecar-missing test              | Assert reopen behavior when sidecar is missing                                                                                                 |
| Node RVF lifecycle test           | Create -> ingest -> query -> compact -> close -> reopen -> query (via `@ruvector/rvf` or `npx ruvector` with `RUVECTOR_BACKEND=rvf`)           |
| WasmBackend expected-throws tests | Assert `open`/`openReadonly` throw with expected code; `deleteByFilter` throws; optional: `compact` no-op, `lineage`/`segments`/`fileId` throw |
| Format future (backlog)           | Track: embed string ID table in RVF meta segment to eliminate sidecar dependency                                                               |


**Files:** `[npm/packages/rvf/src/backend.ts](../../npm/packages/rvf/src/backend.ts)` (~526–537 for WasmBackend throws, sidecar logic in NodeBackend).

### Epic C — Known limitations documentation

Publish honest capability matrix so users and agents know what works where.


| Doc section           | Content                                                                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Node vs WASM table    | Per-operation support (open, query, delete, compact, lineage, segments, deleteByFilter, fileId) with "throws" / "no-op" / "supported"      |
| Default backend       | `ruvector` npm defaults to REDB via `@ruvector/core`; RVF only when `RUVECTOR_BACKEND=rvf` or core missing                                 |
| MCP vs Rust `rvf` CLI | Subset/delta between MCP `rvf_`* tools and Rust `rvf` binary (coordinate with [ADR-A01](./ADR-A01-mcp-cli-surface-alignment.md) docs wave) |
| RuVocal naming        | `rvf.ts` JSON format is not binary RVF — documentation-only clarification                                                                  |
| Hooks vs `.rvf`       | `.ruvector/intelligence.json` memory is not a `.rvf` file — explicit row in matrix                                                         |
| Extensions vs RVF     | `ruvector-extensions` `DatabasePersistence` vs canonical binary RVF                                                                        |
| Rust vs npm rvlite    | Cross-link `crates/rvlite` (Rust, optional rvf-backend feature) vs `npm/packages/rvlite` (JS) to avoid conflation                          |


### Epic B — WASM durability MVP

**Packages:** `npm/packages/rvf`, `npm/packages/rvf-wasm`


| Phase         | Task                                                            | Acceptance                                                                     |
| ------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| B1            | Verify `rvf_store_export` blob is a full RVF file (not partial) | Buffer round-trips: export → reimport → query returns same results             |
| B2            | OPFS or IndexedDB persist + rehydrate + query sample            | Browser test: create store, persist via OPFS, reload page, query succeeds      |
| B3            | Document app-layer durability pattern                           | README shows how to persist WASM store today (export → storage API → reimport) |
| B4 (optional) | Expose durable browser path via `WasmBackend` or dedicated API  | Product decision — only if demand warrants                                     |


### Epic D — ruvector-core migration spike

**Package:** `crates/ruvector-core`


| Task                        | Acceptance                                                                          |
| --------------------------- | ----------------------------------------------------------------------------------- |
| Read spike                  | `ruvector-core` can open and query an existing `.rvf` file alongside its REDB store |
| Dual-write spike (optional) | Writes go to both REDB and RVF; reads from REDB; RVF used for export/interop        |


**File:** `[crates/ruvector-core/src/storage.rs](../../crates/ruvector-core/src/storage.rs)`.

### Future Epics (groomed from research backlog)


| Epic                          | Scope                                                                                                                          | Research source |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------- |
| **F1 — Import + migration**   | `rvf-import` CLI batch validation; mixed-version fleet handling; legacy-to-RVF conversion success criteria                     | §8 interop      |
| **F2 — Ops / FS locking**     | Multi-process locking, concurrent CLI+server open, NFS/cloud-sync, backup/restore, compaction scheduling, corruption detection | §8 ops          |
| **F3 — npm provenance + CI**  | npm provenance for `@ruvector/rvf-node` native addon; targeted WASM SLO CI gating publish                                      | §8 security     |
| **F4 — Adapters audit**       | Verify rvf-adapters (agentdb, agentic-flow, ospipe, rvlite-rust, sona) default store format, tests, doc claims                 | §9 adapters     |
| **F5 — Brain + RuVLLM trace** | mcp-brain-server end-to-end RVF path; ruvllm session/policy vs ADR-029 KV-cache/LoRA/RAG spec                                  | §9 peripheral   |


### Recommended Order

```
Epic A (rvlite honesty)
  |
  +--> Epic A test (on-disk format truth)
  |
  +--> Epic E (sidecar doc + test)
  |          |
  |          +--> Epic C (known limitations docs)
  |                  ^
  |                  |
  +-- Epic B (WASM durability MVP) --+
  
Epic D (core spike) — independent track
```

Epics A and E are prerequisites for Epic C (docs should reflect fixed behavior). Epic B feeds into Epic C (WASM capabilities table). Epic D is independent — can run in parallel.

## Consequences

### Positive

- Users and agents get honest information about what persists where (Epic A + C)
- rvlite `saveToRvf` either works correctly or is clearly documented as JSON-envelope-only (Epic A)
- Browser/edge developers get a documented durability path (Epic B) instead of discovering in-memory limitations at runtime
- ruvector-core migration path is proven before any production switch (Epic D)
- Future epics (F1–F5) are individually trackable instead of a monolithic backlog

### Negative

- Epic A may require a minor version bump of `rvlite` if `getStorageBackend()` return value changes — mitigated by documenting the change as a bug fix (label was always wrong)
- Epic B adds complexity to `WasmBackend` or introduces a parallel API — mitigated by starting with documentation of the manual export/reimport pattern (B3) before any API work (B4)
- Epic D is exploratory effort that may not ship — mitigated by scoping as a branch-only spike with explicit "no prod flip" constraint

### Risks


| Risk                                           | Mitigation                                                                                      |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| rvlite downstream breakage from Epic A         | Fix is documentation + label correction, not API removal; `saveToRvf` still works (writes JSON) |
| WASM OPFS availability                         | OPFS is widely supported (Chrome 102+, Firefox 111+, Safari 15.2+); fallback to IndexedDB       |
| ruvector-core dual-write performance           | Spike measures overhead; dual-write is optional and never the default path                      |
| Sidecar elimination (p1-e3) changes RVF format | Tracked as future format discussion — no format changes in this ADR's scope                     |


## ADR coordination

- **Epic C MCP-vs-Rust-CLI delta** and **ADR-A01 W4 parity matrix**: one canonical doc, the other links. Resolve ownership via `coord-mcp-rvf-doc-canonical` / `dedupe-p2c2-ecosystem-docs` in the plans before writing.
- **CI / ADR SLO**: existing workflows (`build-rvf-node.yml`, `release-rvf-cli.yml`) gate Rust CLI and rvf-node but do not enforce full ADR cold-boot/recall SLOs on every `@ruvector/rvf` npm publish. Document this gap or add targeted CI in Future Epic F3.

## Out of Scope

- **claude-flow memory**, standalone **agentdb** CLI (ADR-029 §5, §8) — out-of-tree; do not block local epics
- **Rust `rvf` CLI vs MCP tool parity** — covered by [ADR-A01](./ADR-A01-mcp-cli-surface-alignment.md)
- **ruvector-extensions** migration to RVF — tracked in Future Epic F1 but not in active waves

## Key Files


| Area                     | File                                                                                           |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| rvlite persistence logic | `[npm/packages/rvlite/src/index.ts](../../npm/packages/rvlite/src/index.ts)`                   |
| Node/WASM backend        | `[npm/packages/rvf/src/backend.ts](../../npm/packages/rvf/src/backend.ts)`                     |
| WASM exports (d.ts)      | `[npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts](../../npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts)`     |
| ruvector-core storage    | `[crates/ruvector-core/src/storage.rs](../../crates/ruvector-core/src/storage.rs)`             |
| RuVocal JSON store       | `[ui/ruvocal/src/lib/server/database/rvf.ts](../../ui/ruvocal/src/lib/server/database/rvf.ts)` |
| RVF types/options        | `[npm/packages/rvf/src/types.ts](../../npm/packages/rvf/src/types.ts)`                         |


## Related Decisions

- **ADR-029**: RVF as canonical binary format — this ADR addresses the implementation gaps against ADR-029's migration table
- **ADR-A01**: MCP/CLI surface alignment — MCP `rvf_`* tool correctness (Wave 0 contract bugs)
- **ADR-001**: Core architecture — ruvector-core REDB storage described here is the Epic D migration source

## Related Plans

- [RVF persistence research todos](../plans/rvf_persistence_research_todos_45f08914.plan.md) — per-todo implementation details for Epics A–D + F1–F5
- [Research streams execution order](../plans/research_streams_execution_order_d0f4d7b3.plan.md) — cross-stream wave sequencing (persistence Epic A is parallel with nomenclature W0)

## Revision History


| Version | Date       | Author    | Changes                                          |
| ------- | ---------- | --------- | ------------------------------------------------ |
| 0.1     | 2026-04-10 | clawdette | Initial proposal from persistence research audit |


