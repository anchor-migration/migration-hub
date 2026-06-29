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
| 12 | [ADR-009](ADR-009-rewrite-engine-presets-and-run-manifest.md) | Rewrite presets, YAML composites, swappable engine port |
| 13 | [ADR-010](ADR-010-anchor-stubborn-integration.md) | **anchor-stubborn** — horizontal LLM context (optional; not a pipeline layer) |
| 14 | [ROADMAP.md](ROADMAP.md) | What is done vs planned |
| 15 | [db-metadata README](https://github.com/anchor-migration/db-metadata) | CLI: `export`, `verify`, `info` |
| — | [anchor-stubborn README](https://github.com/anchor-migration/anchor-stubborn) | SCIP ingest, context weaving, metrics KPI (independent repo) |

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
| **parity-verify** | public | Before/after AST structural diff | Alpha |
| **anchor-explorer** | public | Read-only UI over linked SSOT snapshots | Alpha |
| **anchor-stubborn** | public | SCIP → symbol graph → LLM context stubs ([ADR-010](ADR-010-anchor-stubborn-integration.md); horizontal) | Alpha |
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
├── rewrite-recipes/        OpenRewrite catalog (alpha)
├── parity-verify/          Before/after AST structural diff (alpha)
├── anchor-stubborn/        LLM context compiler (horizontal; ADR-010)
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
OpenRewrite recipes ──► modernized code                 [alpha — stack 3.0–3.3 + L1/L2/L3 + presets]
classify-lists (on-demand) ──► L2 gate JSON             [alpha — ADR-008 M2]
verify / parity ──► proof                               [alpha — parity-verify]
anchor-stubborn ──► optional LLM context (SCIP → stubs) [alpha — ADR-010; not in SSOT path]
```

Details: [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md), [ARCHITECTURE.md](ARCHITECTURE.md), [ADR-010](ADR-010-anchor-stubborn-integration.md).

## What is verified today

| Capability | Status |
|------------|--------|
| `db-metadata` export + verify (SQLite source) | ✅ pytest + manual |
| Duke's Bank MySQL Docker demo | ✅ |
| `java-ast-ssot` export on bank module | ✅ Alpha |
| `java-ast-ssot crosswalk` (code + schema SSOT) | ✅ Alpha |
| `java-ast-ssot classify-lists` (homogeneous / tuple / unknown) | ✅ Alpha |
| Duke's Bank E2E (linked.db + anchor-explorer) | ✅ Verified |
| `rewrite-recipes` harness + smoke (3.0) | ✅ 16 tests |
| Session→Service + CMP→JPA on Duke's Bank fixtures (3.1–3.3) | ✅ |
| Language modernization L1 + L2 (ADR-008 M1/M3) | ✅ |
| Language modernization L3 tuple proposals (ADR-008 M4) | ✅ |
| Preset catalog ADR-009 (`Smoke`, `LanguageL1Only`, `LanguageL2Only`, `LanguageL3Only`, `DukesBankStackMigration`) | ✅ |

## Program progress snapshot

| Area | Latest milestone |
|------|------------------|
| Schema SSOT | `db-metadata` alpha; Duke's Bank verified |
| Code SSOT | `java-ast-ssot` 1.0 — core + `javaee-ejb2-jboss` + `jpa` + **`mybatis`** + crosswalk |
| List analysis | `classify-lists` — ephemeral JSON, no SQLite sidecar |
| Human review | `anchor-explorer` alpha — crosswalk graph |
| Stack rewrite | Session `BeanState` → Spring service; scalar CMP → JPA |
| Language rewrite | L1 mechanical swaps; L2 homogeneous `ArrayList<E>`; **L3 tuple → result class** |
| Orchestration | ADR-009 YAML presets + `anchor.rewrite.preset` property |
| Parity | `parity-verify` v0.1 — before/after AST stable-ID diff (JSON) |
| LLM context | `anchor-stubborn` v0.3 — metrics KPI; ~86% token savings on demo-spring `OrderService` |
| **Next** | JPA Duke's Bank E2E re-export (4d); parity-verify behavioral tests + HTML report |

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

1. **JPA Duke's Bank E2E re-export** after CMP→JPA; **parity-verify** behavioral matrix + HTML report
2. Blog draft from lab-notes backlog (private)

**Done:** ADR-004 Step 5 `mybatis` profile + crosswalk (`resultMap`, JOIN `read_model`); ADR-004 Step 4 `jpa` profile + crosswalk (`@Entity` / `@Table` / `@Column`); ADR-008 M4 L3 — [tuple-list-l3.md](https://github.com/anchor-migration/rewrite-recipes/blob/main/docs/tuple-list-l3.md); ADR-008 M3 L2 — [homogeneous-raw-list-l2.md](https://github.com/anchor-migration/rewrite-recipes/blob/main/docs/homogeneous-raw-list-l2.md); ADR-008 M2 `classify-lists` — [list-usage-classifier.md](https://github.com/anchor-migration/java-ast-ssot/blob/main/docs/list-usage-classifier.md); ADR-009 preset manifests — [rewrite-presets.md](https://github.com/anchor-migration/rewrite-recipes/blob/main/docs/rewrite-presets.md); ADR-008 L1 — [vector-to-arraylist-l1.md](https://github.com/anchor-migration/rewrite-recipes/blob/main/docs/vector-to-arraylist-l1.md); 3.3 CMP→JPA scalar `AccountBean` — [cmp-scalar-entity-to-jpa-account-bean.md](https://github.com/anchor-migration/rewrite-recipes/blob/main/docs/cmp-scalar-entity-to-jpa-account-bean.md); 3.2 Session→Service chain — [session-bean-to-spring-service-account-controller.md](https://github.com/anchor-migration/rewrite-recipes/blob/main/docs/session-bean-to-spring-service-account-controller.md); 3.0 harness — `.\scripts\run-test.ps1`; Duke's Bank E2E — [DUKESBANK-DEMO.md#e2e-quick-path](DUKESBANK-DEMO.md#e2e-quick-path)

See **[AGENTS.md](../AGENTS.md)** at the root of this repository for session bootstrap instructions.

When the user says “continue Anchor Migration”, read in order:

1. This file  
2. `AGENTS.md`  
3. Latest `lab-notes/journal/*-session-wrapup.md` (if available)  
4. `DUKESBANK-DEMO.md` if work touches Duke's Bank or code SSOT  

Do **not** assume chat history from prior sessions exists.
