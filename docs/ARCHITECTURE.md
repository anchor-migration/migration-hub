# Architecture

## Overview

Anchor Migration is a **multi-repo program** for legacy Java modernization. Each repository owns one stage of the pipeline. Shared contracts (SQLite schemas, entity key formats, pattern IDs) link the stages without tight coupling.

## Layers

### 1. Extraction → SSOT

| Input | Tool | Output |
|-------|------|--------|
| Live relational database | `db-metadata` | Schema SSOT (SQLite): tables, columns, PK, FK, indexes |
| Legacy Java codebase | `java-ast-ssot` | **Core:** Java AST SSOT (types, methods, fields, imports). **Optional profiles:** stack adapters (e.g. `javaee-ejb2-jboss` for EJB/XML). |

Both exporters are **read-only** on source systems and produce **versioned snapshots** suitable for drift comparison.

#### Language-specific AST repos

Extractors are named **`{language}-ast-ssot`**, not a single generic code repo. `java-ast-ssot` owns **Java source structure**; legacy **stack** details (Java EE, Spring, JPA) are **profiles** inside that repo — see [ADR-002](docs/ADR-002-java-ast-ssot-core-and-profiles.md).

Future heterogeneous migration (e.g. COBOL → Java) may add `cobol-ast-ssot`; cross-language linking is a separate layer on top of per-language SSOTs.

#### Duke's Bank

Reference **demo application**, not the boundary of `java-ast-ssot`. It validates the first profile (`javaee-ejb2-jboss`) plus `db-metadata` crosswalk.

### 2. Transformation

| Input | Tool | Output |
|-------|------|--------|
| Code SSOT + Schema SSOT + pattern ID | `rewrite-recipes` | OpenRewrite recipes tuned to detected patterns |
| Recipes + source tree | OpenRewrite CLI / Maven plugin | Refactored source |

Schema SSOT informs entity/table mapping (e.g. `@Table`, `@Column`, relationship cardinality). Code SSOT informs safe, scoped recipe application.

AI-assisted refactoring handles non-mechanical cases; outputs remain subject to parity verification.

### 3. Verification

| Input | Tool | Output |
|-------|------|--------|
| Old + new codebases | `parity-verify` | Parity report: AST diffs, behavioral test matrix, gaps |
| Schema SSOT | `db-metadata verify` | Export reconciliation against live DB |

Verification is not optional in the intended workflow: migrate → verify → fix → re-verify.

### 4. Human interface — SSOT Explorer

| Input | Tool | Output |
|-------|------|--------|
| Schema + code + linked SSOT (SQLite) | **`ssot-explorer`** | Interactive graphs: schema ER, code structure, crosswalk with **edge coloring** |

Explorer is **read-only** and lives in a **separate repo**. It is not optional glue for demos: as long as humans participate in migration review, Explorer is a **first-class interface** with proper architecture and UX — see [ADR-005](docs/ADR-005-multi-tier-alignment-and-ssot-explorer.md).

Mapping quality uses **bidirectional edge colors** on each link: green/yellow/orange/red **per direction** (traffic-map model) — see [ADR-005](docs/ADR-005-multi-tier-alignment-and-ssot-explorer.md).

## Data contracts

See [SSOT-SCHEMA.md](SSOT-SCHEMA.md) for cross-repo schema versioning and entity key conventions.

### Schema SSOT (implemented)

- Repository: `db-metadata`
- Format: SQLite v1 (`export_run`, `db_table`, `db_column`, `db_foreign_key`, …)
- Stable IDs: `schema.table.column`

### Java AST SSOT (alpha — core + profiles)

- Repository: `java-ast-ssot`
- **Core format:** SQLite v1 — `export_run`, `java_type`, `java_method`, `java_field`, `java_import`
- **Profile (v0.1):** `javaee-ejb2-jboss` — EJB/JBoss XML tables and crosswalk edges; Duke's Bank validated
- Design: [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md) (core vs profiles), [ADR-003](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md) (AST sidecars vs LST), [ADR-004](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md) (crosswalk)
- Stable IDs (core): e.g. `com.example.Foo#bar(int,String)`

## Pattern catalog

`pattern-catalog` documents **migration patterns** (EJB → Spring, DAO → Repository, etc.):

- Detection heuristics (AST + schema signals)
- Linked OpenRewrite recipes
- Verification checklist
- Example before/after projects

Patterns are the bridge between heterogeneous legacy code and reusable automation.

## AI-assisted development role

This organization is built with **AI as the primary coding partner**. The human developer defines architecture and [boundary protocols](DEVELOPMENT-MODEL.md); AI implements features, tests, and docs against those contracts.

See **[Development model](DEVELOPMENT-MODEL.md)** for the full operating philosophy.

### AI in the migration pipeline (building the toolkit)

- Scaffolding repos, modules, tests, and documentation
- Implementing dialect adapters, verification SQL, CLI commands
- Drafting OpenRewrite recipes and parity test matrices for human review

### AI in the product (optional, bounded)

Optional **self-healing or exploration nodes** may suggest fixes or additional tests when verification fails. These are advisory only:

- proposals become deterministic code or recipes after human approval
- the critical path (export → transform → verify) remains LLM-free at runtime

### Hard rules

AI does **not** replace SSOT or verification. Generated changes must pass mechanical checks and parity gates. Shipped code is **100% deterministic Python or Java** — reproducible, diffable, and CI-gated.

## Repository independence

Each repo has its own:

- Language / build tooling
- CI pipeline
- Release cadence

Integration is via **file artifacts** (SQLite snapshots) and documented contracts, not shared libraries (initially).

## Future considerations

- Unified CLI orchestrating export → rewrite → verify
- Additional `{language}-ast-ssot` repos for heterogeneous legacy (e.g. COBOL)
- `demo-legacy-app` sample project for end-to-end demos
- Org-level GitHub Actions for cross-repo smoke tests
