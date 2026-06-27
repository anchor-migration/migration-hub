# Start here — Anchor Migration

Program map for humans and for onboarding AI in a new session. Read this file first, then follow links for depth.

## What this program is

**Anchor Migration** is an SSOT-driven toolkit for **legacy Java modernization**, built with **architecture-led, AI-assisted development**:

- The **developer** defines architecture and boundary protocols.
- **AI** implements most code under those contracts.
- **Shipped artifacts** are deterministic Python and Java (same inputs → same outputs).

Public showcase: https://github.com/anchor-migration

## Reading order (recommended)

| Step | Document | Why |
|------|----------|-----|
| 1 | [DEVELOPMENT-MODEL.md](DEVELOPMENT-MODEL.md) | How we build; deterministic core vs optional AI nodes |
| 2 | [ARCHITECTURE.md](ARCHITECTURE.md) | Layers + **architecture diagrams** (Mermaid) |
| 3 | [SSOT-SCHEMA.md](SSOT-SCHEMA.md) | Cross-repo contracts, stable IDs |
| 4 | [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md) | Reference demo; DRG design, AST/LST/XML decisions |
| 5 | [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md) | `java-ast-ssot` core vs stack profiles |
| 6 | [ADR-003](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md) | AST + sidecars vs LST at rewrite time |
| 7 | [ADR-004](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md) | Code ↔ schema crosswalk; mapping roles |
| 8 | [ADR-005](ADR-005-multi-tier-alignment-and-ssot-explorer.md) | Multi-tier alignment, edge coloring, Anchor Explorer |
| 9 | [ADR-006](ADR-006-multi-role-decision-review.md) | Multi-role decision review before implementation |
| 10 | [ADR-007](ADR-007-rewrite-recipes-session-and-cmp-jpa.md) | rewrite-recipes: Session→Service (BeanState) vs CMP→JPA |
| 11 | [ADR-008](ADR-008-java-language-modernization-and-tuple-lists.md) | Language modernization: generics, Vector, tuple lists → result classes |
| 12 | [ROADMAP.md](ROADMAP.md) | What is done vs planned |
| 12 | [db-metadata README](https://github.com/anchor-migration/db-metadata) | CLI: `export`, `verify`, `info` |

**Private (if you have access):** `lab-notes/journal/2026-06-27-session-wrapup.md` — latest session log.

## Repository map

| Repo | Visibility | Role | Status |
|------|------------|------|--------|
| **migration-hub** | public | Program docs (this repo) | Active |
| **db-metadata** | public | Live DB → schema SSOT (SQLite) | Alpha |
| **.github** | public | Org profile README | Active |
| **lab-notes** | **private** | Journals, ADR drafts, blog drafts | Active |
| **java-ast-ssot** | public | Java AST SSOT (core + optional stack profiles) | Alpha |
| **rewrite-recipes** | public | OpenRewrite catalog | Alpha |
| **parity-verify** | public | Old vs new parity | Planned |
| **anchor-explorer** | public | Read-only UI over linked SSOT snapshots | Alpha |
| **pattern-catalog** | public | Migration patterns | Planned |

## Local workspace (author machine)

```
C:\github\anchor-migration\
├── migration-hub/
├── db-metadata/
├── lab-notes/              → github.com/anchor-migration/anchor-migration-lab-notes (private)
├── demo-dukesbank/         MySQL bridge for Duke's Bank demo
├── java-ast-ssot/          Java AST SSOT exporter
├── anchor-explorer/        Read-only crosswalk UI (React + Vite)
└── anchor-migration.code-workspace

C:\github\dukesbank\        Duke's Bank legacy sample (external to org)
```

## Pipeline (one picture)

See **[architecture diagrams](ARCHITECTURE.md#program-overview)** in ARCHITECTURE.md for full Mermaid maps (repos, crosswalk, tiers).

```
Live MySQL ──db-metadata──► schema SSOT (SQLite)
Legacy Java + XML ──java-ast-ssot──► Java AST SSOT (SQLite)
         └─ crosswalk ──► linked SSOT (code_schema_link + edge colors)
anchor-explorer ──► human UI (read-only)               [alpha]
OpenRewrite recipes ──► modernized code                 [planned]
verify / parity ──► proof                               [planned]
```

Details: [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md), [ARCHITECTURE.md](ARCHITECTURE.md).

## What is verified today

| Capability | Status |
|------------|--------|
| `db-metadata` export + verify (SQLite source) | ✅ pytest + manual |
| Duke's Bank MySQL Docker demo | ✅ |
| `java-ast-ssot` export on bank module | ✅ Alpha |
| `java-ast-ssot crosswalk` (code + schema SSOT) | ✅ Alpha |
| Duke's Bank E2E (linked.db + anchor-explorer) | ✅ Verified |

## Language-specific AST repos

Extractors use **`{language}-ast-ssot`** names. **`java-ast-ssot`** = generic Java core + optional profiles (Duke's Bank validates `javaee-ejb2-jboss`). Future **`cobol-ast-ssot`** etc. for heterogeneous migration.

## Conventions (do not forget)

| Topic | Rule |
|-------|------|
| Chat with AI | Chinese is fine |
| Code comments, docs, commits | **English** |
| SSOT | Live DB + live code/XML — not LLM memory |
| Comments in code SSOT | Separate layer; no forced semantic binding v1 |
| Code parse for SSOT | JavaParser + XML; OpenRewrite LST only for rewriting |
| Public vs draft | Stabilized docs → here; raw thinking → lab-notes (private) |

## Next work (priority)

1. **`rewrite-recipes` 3.1a** — CMP→JPA capability matrix on Duke's Bank ([ADR-007](ADR-007-rewrite-recipes-session-and-cmp-jpa.md)); BeanState session recipe before scalar CMP→JPA
2. **`jpa` / `mybatis` profiles** — ADR-004 Step 4–5
3. Blog draft from lab-notes backlog (private)

**Done:** `rewrite-recipes` 3.0 harness — https://github.com/anchor-migration/rewrite-recipes (`.\scripts\run-test.ps1` / Docker); Duke's Bank E2E — [DUKESBANK-DEMO.md#e2e-quick-path](DUKESBANK-DEMO.md#e2e-quick-path) + `demo-dukesbank/scripts/run-e2e.ps1`

See **[AGENTS.md](../AGENTS.md)** at the root of this repository for session bootstrap instructions.

When the user says “continue Anchor Migration”, read in order:

1. This file  
2. `AGENTS.md`  
3. Latest `lab-notes/journal/*-session-wrapup.md` (if available)  
4. `DUKESBANK-DEMO.md` if work touches Duke's Bank or code SSOT  

Do **not** assume chat history from prior sessions exists.
