# RVF / RuVector persistence layer — research deliverable

**Scope:** Single-session research (no subagents). **Do not treat this file as ADR-029**; it inventories code and docs as of the research date.

## 1. Executive summary

Canonical **binary `.rvf` on disk** is **fully wired** for **Node.js** via `@ruvector/rvf` → `@ruvector/rvf-node` (N-API) → `rvf-runtime::RvfStore`. **WASM** (`WasmBackend` + `@ruvector/rvf-wasm`) provides an **in-memory** store handle with optional **buffer** `rvf_store_open` / `rvf_store_export` — not file-backed paths in the TypeScript SDK. **npm `rvlite`** advertises RVF when `@ruvector/rvf-wasm` is installed but `**writeRvf` / `readRvf` do not exist** on that package; `saveToRvf` / `loadFromRvf` fall back to a **JSON envelope** (`magic: "RVF1"`), which is **not** ADR-029 binary RVF. `**ruvector-core`** remains **REDB + memmap + rkyv** (no `rvf` references). **RuVocal `rvf.ts`** is a **JSON document store** (`format: "rvf-database"`), not binary RVF.

---

## 2. Persistence matrix (runtime × surface)


| Package / surface                                            | Runtime                 | Durable default?        | Canonical binary `.rvf`?                           | Format / notes                                                                                                         | Peripheral role                                              |
| ------------------------------------------------------------ | ----------------------- | ----------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `crates/rvf` (`rvf-runtime`, `rvf-cli`)                      | Native Rust             | Yes (file)              | Yes                                                | Segmented RVF; integration tests in `crates/rvf/tests/`                                                                | Core format implementation                                   |
| `@ruvector/rvf` `NodeBackend`                                | Node                    | Yes (path)              | Yes                                                | N-API → `RvfStore`; string IDs ↔ numeric labels + **sidecar JSON** (`*.mappings.json` pattern in `backend.ts`)         | Primary TS disk path                                         |
| `@ruvector/rvf` `WasmBackend`                                | Node/browser            | No (RAM)                | No                                                 | `open`/`openReadonly` throw; `create` uses `rvf_store_create`; no lineage/kernel/segments                              | TS SDK limitation                                            |
| `@ruvector/rvf-wasm`                                         | WASM                    | Via app (buffer)        | Partial                                            | `rvf_store_open(buf,len)`, `rvf_store_export` — **no** `writeRvf`/`readRvf` path helpers                               | Microkernel + in-wasm store                                  |
| `npm/packages/rvlite` default `createRvLite`                 | Node/browser            | IndexedDB / memory      | **No**                                             | WASM `RvLite`; `getStorageBackend()==='rvf'` if package resolvable — **does not** wire `rvf_store_*` into default init | Misleading label vs behavior                                 |
| `rvlite` `saveToRvf` / `loadFromRvf`                         | Node only               | File                    | **Usually no**                                     | Checks `writeRvf`/`readRvf` on wasm module → **always absent** → JSON `RVF1` envelope                                  | Contract mismatch                                            |
| `ruvector` npm (`@ruvector/core` path)                       | Node                    | Yes                     | No                                                 | REDB layout per architecture docs                                                                                      | Default when native installs                                 |
| `ruvector` + `RUVECTOR_BACKEND=rvf`                          | Node                    | Yes                     | Yes                                                | Loads `@ruvector/rvf`                                                                                                  | Opt-in                                                       |
| `ruvector` MCP `rvf_*` tools                                 | Node                    | Yes (paths)             | Yes (delegates to RVF stack)                       | ~9 tools: create, open, ingest, query, delete, status, compact, derive, segments, examples                             | Subset of Rust `rvf` CLI (~18 subcommands in package README) |
| `ruvector-extensions` `DatabasePersistence`                  | Node                    | Yes                     | No                                                 | JSON / binary / sqlite **framework** for JS `VectorDB`                                                                 | Parallel persistence story                                   |
| `crates/ruvector-core`                                       | Native Rust             | Yes                     | No                                                 | `storage.rs` + redb                                                                                                    | ADR-029 migration **not** done                               |
| `crates/rvlite` (Rust)                                       | Native / WASM features  | Varies                  | Optional `rvf-backend` feature (per rvlite README) | Separate from npm rvlite                                                                                               | Adapter: `rvf-adapters/rvlite`                               |
| RuVocal `ui/ruvocal/.../rvf.ts`                              | Node (SvelteKit server) | Yes (JSON file)         | No                                                 | `rvf_version: "2.0"`, `format: "rvf-database"`                                                                         | Naming collision risk                                        |
| `crates/mcp-brain-server`                                    | Server                  | Varies                  | Optional / federation                              | `rvf_bytes`, `rvf_gcs_path`, `rvf_federation`, witness flags                                                           | RVF-adjacent memory path                                     |
| `crates/ruvllm`                                              | Native Rust             | Yes                     | No                                                 | `AgenticDB` + `DbOptions.storage_path`                                                                                 | ADR-029 RuVLLM section **spec vs code**                      |
| `rvf-adapters` (agentdb, agentic-flow, ospipe, rvlite, sona) | Rust                    | Per adapter             | Yes (RVF-backed where implemented)                 | In-repo integration surfaces                                                                                           | Partial ADR row coverage                                     |
| `npm/agentic-integration`, `ruvector-router-core`            | —                       | If any, not RVF-labeled | No                                                 | No `rvf` grep hits                                                                                                     | Routing indexes TBD                                          |
| Hooks / `.ruvector` intelligence                             | Node                    | Per hook config         | No                                                 | rvlite/json backends                                                                                                   | User-facing “memory” ≠ `.rvf` file                           |


---

## 3. rvlite contract audit (`saveToRvf` / `loadFromRvf`)

**Claim in code:** Prefer `@ruvector/rvf-wasm` native binary reader/writer when `writeRvf` / `readRvf` exist.

**Fact:** `[npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts](../../../npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts)` exports `rvf_store_*`, `rvf_alloc`, segment parse, witness helpers — **no** `writeRvf`, `readRvf`, or filesystem APIs.

**Result:** For Node, `saveToRvf` / `loadFromRvf` **always** use the JSON envelope path when `@ruvector/rvf-wasm` is present (the `if (this.rvfModule && typeof this.rvfModule.writeRvf === 'function')` branch is **dead**).

`**getStorageBackend()`:** Returns `'rvf'` when `require.resolve('@ruvector/rvf-wasm')` succeeds, but `**RvLite.init()`** does not pass RVF wasm store into the main `RvLite` WASM constructor — so the string **overstates** persistence to canonical RVF.

---

## 4. WasmBackend vs `@ruvector/rvf-wasm` (durability)


| Capability                      | NodeBackend              | WasmBackend                                                                                                                                                                                        |
| ------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| File create/open                | Yes                      | `create` only; `open` throws                                                                                                                                                                       |
| `deleteByFilter`                | Yes (native)             | **Throws** — not supported                                                                                                                                                                         |
| `compact`                       | Yes                      | No-op zeros                                                                                                                                                                                        |
| Lineage (`fileId`, `derive`, …) | Yes                      | Throws                                                                                                                                                                                             |
| Kernel / eBPF / `segments`      | Yes                      | Throws                                                                                                                                                                                             |
| IDs                             | String + sidecar mapping | **BigInt** numeric strings                                                                                                                                                                         |
| Low-level WASM                  | —                        | `rvf_store_export` / `rvf_store_open(buffer)` **could** support durability if JS writes bytes to **OPFS**, **IndexedDB**, or **File System Access** — **not implemented** in `@ruvector/rvf` today |


**Outline for a supported browser story (research recommendation only):**

1. After mutations, call `rvf_store_export` into a `Uint8Array`, persist to OPFS or IndexedDB chunking.
2. On load, read bytes and call `rvf_store_open(ptr, len)` (or recreate + ingest if export format is store blob, not full RVF file — **verify** against `rvf-wasm` implementation).
3. Document that this is **app-layer** durability until `WasmBackend` exposes `open(path)` on environments with a virtual FS.

---

## 5. ADR-029 migration table vs repository


| ADR-029 row                  | In-repo status (high level)                                                                                                                           |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ruvector-core** REDB → RVF | **Not migrated**; core uses redb/memmap/rkyv.                                                                                                         |
| **agentdb**                  | **In-repo adapter:** `crates/rvf/rvf-adapters/agentdb` (`RvfStore` vector store). Separate `npx agentdb` product **out of tree** — not verified here. |
| **claude-flow memory**       | **Out of tree** — scope external.                                                                                                                     |
| **agentic-flow**             | **In-repo adapter:** `rvf-adapters/agentic-flow`.                                                                                                     |
| **ospipe**                   | **In-repo adapter:** `rvf-adapters/ospipe`.                                                                                                           |
| **rvlite**                   | **Rust adapter** + **npm rvlite** (JSON envelope / WASM disconnect as above).                                                                         |
| **sona**                     | **In-repo adapter:** `rvf-adapters/sona`.                                                                                                             |


---

## 6. Integration tests — existing and recommended

### Existing (run for regression)

- `npm/packages/rvf/tests/` — e.g. `test-id-mapping.js` (sidecar mappings).
- `crates/rvf/tests/rvf-integration/` — crash safety, progressive recall, wire round-trip, unknown segment preservation, etc.
- `.github/workflows/build-rvf-node.yml`, `release-rvf-cli.yml`, `sync-rvf-examples.yml` — **Rust CLI and rvf-node** CI; **not** a full ADR cold-boot/recall SLO gate on every npm publish.

### Recommended additions (research backlog)

1. **Node RVF lifecycle:** script or test: create → ingest → query → compact → close → reopen → query (via `@ruvector/rvf` or `npx ruvector` with `RUVECTOR_BACKEND=rvf`).
2. **rvlite persistence truth:** assert file bytes are **not** valid binary RVF magic when using default `saveToRvf` JSON path, or fix implementation and assert binary.
3. **Sidecar JSON:** copy `.rvf` without sidecar; reopen with string IDs — document failure mode; optional automated test.
4. **WasmBackend:** test that `open` throws with expected code; `deleteByFilter` throws.

---

## 7. Naming risk: RuVocal `rvf.ts` vs binary RVF

**Recommendation:** **Documentation-only** fix in product/docs: state clearly that **“RVF database”** in RuVocal is a **JSON application format** for Mongo compatibility, **not** the RuVector Format binary container (ADR-029). Renaming types/files is higher churn; defer unless user confusion is measured.

---

## 8. Tier D / E extensions (interop, ops, security, adjacent)


| Theme                       | Research notes                                                                                                                                                                                                                                                                     |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Interop & migration**     | Need explicit **ruvector-core → RVF** and **ruvector-extensions → RVF** stories; `rvf-import` crate for CSV/JSON; define validation + mixed-version semantics.                                                                                                                     |
| **Ops & FS**                | File locking, multi-process open, NFS/cloud, backup/restore, compaction policy, corruption detection, metrics — **not** specified in npm SDK layer.                                                                                                                                |
| **API parity**              | Maintain a **known limitations** table (Section 4) for Node vs WASM.                                                                                                                                                                                                               |
| **Security & supply chain** | At-rest encryption not standardized; signing optional in runtime; `build-rvf-node.yml` builds native addon — map to **npm provenance** separately; ADR SLOs (e.g. WASM 8 KB tile) require **targeted CI** — not observed as gating **all** `@ruvector/rvf` publishes in this scan. |
| **MCP vs CLI**              | MCP exposes a **subset** of Rust `rvf` CLI (no `inspect`, `ingest` file CLI parity in MCP naming, etc.) — document delta.                                                                                                                                                          |
| **Out-of-tree**             | claude-flow, standalone agentdb CLI: **explicitly out of scope** for this repo’s verification unless submodule added.                                                                                                                                                              |


---

## 9. Peripheral modules (embeddings, brain, LLM, routing)


| Module                                | Persistence / RVF relation                                                                                      |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **mcp-brain-server**                  | Strong RVF **federation** and optional **RVF bytes** on memory types; trace embed → store → optional container. |
| **ruvllm**                            | **AgenticDB** today; ADR-029 RuVLLM bullets (KV cache, LoRA in RVF) = **gap**.                                  |
| **rvf-adapters/**                     | Primary in-repo mapping for agentdb, agentic-flow, ospipe, rvlite, sona.                                        |
| **agentic-integration / router-core** | No direct RVF; include in matrix if local vector indexes are added later.                                       |
| **Hooks / `.ruvector`**               | Parallel “intelligence persistence” story.                                                                      |


---

## 10. Gap list (owners / areas)


| Gap                                            | Suggested owner area                               |
| ---------------------------------------------- | -------------------------------------------------- |
| Dead `writeRvf`/`readRvf` checks in npm rvlite | `npm/packages/rvlite`                              |
| `getStorageBackend()` semantics                | `npm/packages/rvlite`                              |
| WasmBackend file / OPFS durability             | `npm/packages/rvf` + `@ruvector/rvf-wasm` bindings |
| ruvector-core vs ADR-029                       | `crates/ruvector-core` + migration design          |
| MCP vs full `rvf` CLI                          | `npm/packages/ruvector/bin/mcp-server.js` + docs   |
| RuVocal naming                                 | `ui/ruvocal` docs / comments                       |
| rvlite README claims vs code                   | `npm/packages/rvlite/README.md`                    |
| RuVLLM vs ADR-029                              | `crates/ruvllm`                                    |


---

## 11. Recommended epics (with test criteria)

1. **Epic A — rvlite honesty**
  - **Done when:** README and JSDoc match behavior; either implement thin Node binary path using `rvf_store_export` + fs **or** rename envelope to non-“RVF binary” wording; tests prove on-disk format.
2. **Epic B — Wasm durability MVP**
  - **Done when:** One documented path (OPFS or IndexedDB) persists store bytes; sample or test rehydrates and queries.
3. **Epic C — Known limitations doc**
  - **Done when:** Published doc (or README section) = Section 4 table + MCP/CLI delta.
4. **Epic D — ruvector-core migration spike**
  - **Done when:** ADR or spike branch proves read path from RVF or dual-write strategy; no production flip required.
5. **Epic E — Sidecar robustness**
  - **Done when:** Documented + optional test for `.rvf` without sidecar; consider embedding string table in RVF meta (future format discussion).

---

## 12. References (in-repo)

- [ADR-029](../../adr/ADR-029-rvf-canonical-format.md)  
- `[npm/packages/rvf/src/backend.ts](../../../npm/packages/rvf/src/backend.ts)`  
- `[npm/packages/rvlite/src/index.ts](../../../npm/packages/rvlite/src/index.ts)`  
- `[npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts](../../../npm/packages/rvf-wasm/pkg/rvf_wasm.d.ts)`  
- `[crates/ruvector-core/src/storage.rs](../../../crates/ruvector-core/src/storage.rs)`  
- `[ui/ruvocal/src/lib/server/database/rvf.ts](../../../ui/ruvocal/src/lib/server/database/rvf.ts)`