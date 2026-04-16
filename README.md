# ruvector_surface_discovery

**CLI/MCP alignment and RVF persistence toolkit.**

> Build yourself — results will vary based on your own models and techniques.

---

## Overview

This workspace is the research and planning surface for the RuVector ecosystem. It captures the current state of two interrelated problems:

1. **MCP/CLI surface alignment** — the `ruvector` npm package exposes two parallel interfaces (MCP server and Commander CLI) with contract bugs, parity gaps, and structural drift
2. **RVF persistence layer completeness** — the RVF (RuVector Format) binary specification is not fully wired across all runtime paths (Node, WASM, rvlite, ruvector-core)

Everything here is research output. The deliverables are **ADRs**, **analysis documents**, and **implementation plans** — not shipped production code.

---

## Repository Structure

```
ruvector_surface_discovery/
├── adr/                          # Architecture Decision Records
│   ├── ADR-A01-mcp-cli-surface-alignment.md
│   └── ADR-A02-rvf-persistence-layer-completion.md
│
├── analysis/
│   ├── rvf/                       # RVF format research
│   │   ├── INDEX.md
│   │   └── RVF_PERSISTENCE_ECOSYSTEM_RESEARCH.md
│   └── ruvector/                  # RuVector ecosystem research
│       ├── README.md
│       ├── CLI_MCP_ECOSYSTEM_RESEARCH.md
│       └── CLI_MCP_NOMENCLATURE_RESEARCH.md
│
└── plans/                         # Implementation plans (per-ADR epic分解)
```

---

## Key Documents

### Architecture Decision Records

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-A01](./adr/ADR-A01-mcp-cli-surface-alignment.md) | MCP/CLI Surface Alignment | Proposed |
| [ADR-A02](./adr/ADR-A02-rvf-persistence-layer-completion.md) | RVF Persistence Layer Completion | Proposed |

### ADR-A01 — MCP/CLI Surface Alignment

Fixes three categories of problems between the MCP server (`bin/mcp-server.js`, 95 tools across 9 domains) and the CLI (`bin/cli.js`):

- **Contract bugs** — silent failures from type mismatches (e.g., `rvf_create` dimension/dimensions, `rvf_delete` ids number vs string)
- **Parity gaps** — capabilities that exist on one surface only (e.g., CLI `rvf delete`, MCP `workers_cancel`)
- **Structural drift** — no shared source of truth between MCP tool schemas and CLI flag definitions

Shipped in waves: W0 contract fixes → W1 feature parity → W2 schema hardening → W3 manifest + codegen → W4 documentation.

### ADR-A02 — RVF Persistence Layer Completion

Documents which RuVector runtime paths actually persist canonical `.rvf` files vs. claiming to:

| Surface | Canonical `.rvf` | Actual Status |
|---------|-----------------|---------------|
| Rust `crates/rvf/` | Yes | Full implementation + tests |
| Node `@ruvector/rvf-node` N-API | Yes | Full implementation |
| WASM `WasmBackend` | No | In-memory only, `open()` throws |
| rvlite npm | Broken | `saveToRvf` writes JSON, not binary RVF |
| ruvector-core Rust | No | REDB storage, not RVF |

Epic order: A (rvlite honesty) → E (sidecar + tests) → C (known limitations docs) → B (WASM durability) → D (ruvector-core spike).

### Analysis Documents

| Document | Scope |
|----------|-------|
| [RVF index](./analysis/rvf/INDEX.md) | Links to the full RVF specification (10 specs + wire format + microkernel + domain profiles + benchmarks) |
| [CLI/MCP ecosystem research](./analysis/ruvector/CLI_MCP_ECOSYSTEM_RESEARCH.md) | Parity matrix, golden paths, test gaps, epic ordering |
| [CLI/MCP nomenclature](./analysis/ruvector/CLI_MCP_NOMENCLATURE_RESEARCH.md) | Per-tool schema/handler/CLI/key alignment, error ergonomics |

---

## Build Yourself

This workspace contains research artifacts, not runnable code. To build and run the actual RuVector packages:

```bash
# Clone the monorepo
git clone https://github.com/ruvio/ruvector

# Install dependencies
npm install

# Build the RVF npm packages
cd npm/packages/rvf && npm run build

# Use the CLI
npx ruvector --help

# Use the MCP server (stdio)
node bin/mcp-server.js
```

**Results will vary** based on your own models, runtime environment, and techniques. The ADRs and analysis here are the truth surface — actual behavior may differ from what is documented until the epics in these ADRs are implemented.

---

## Background: RVF Format

RVF (RuVector Format) is a self-reorganizing append-only binary vector format with:

- **Append-only + compaction** (not random writes)
- **Two-level manifest system** with hotset pointers and progressive boot
- **Progressive indexing** — Layer A/B/C availability, lazy build, partial search
- **Temperature tiering** — adaptive layout, access sketches, promotion/demotion
- **SIMD-aligned queries** — 384-dim fp16 L2 in ~12 AVX-512 cycles
- **ML-DSA-65 quantum signatures** — 3,309 B signatures, ~4,500 sign/s

See [RVF index](./analysis/rvf/INDEX.md) for the full 10-specification document tree.

---

## Related Repositories

| Repo | Description |
|------|-------------|
| `ruv.io/ruvector` | Main monorepo (npm packages + Rust crates) |
| `ruv.io/rvf` | RVF binary format specification |
| `ruv.io/rvlite-rust` | Rust rvlite with optional rvf-backend feature |

---

## Credits & Legal

This project is derived from [RuVector by Ruven Cohen](https://github.com/ruvnet/ruvector), a high-performance, real-time, self-learning vector GNN memory database built in Rust (v2.1.x).

The research and analysis in this workspace identifies gaps and proposes improvements to the MCP/CLI surface alignment and RVF persistence layer of the RuVector ecosystem. It is based on code within the RuVector monorepo; behavior of your own build will vary based on your models, runtime environment, and techniques.

See [LICENSE](./LICENSE) for the full MIT license terms.

---

*Last updated: 2026-04-16*
