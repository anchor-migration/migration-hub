# Architecture

## Overview

Anchor Migration is a **multi-repo program** for legacy Java modernization. Each repository owns one stage of the pipeline. Shared contracts (SQLite schemas, entity key formats, pattern IDs) link the stages without tight coupling.

## Layers

### 1. Extraction → SSOT

| Input | Tool | Output |
|-------|------|--------|
| Live relational database | `db-metadata` | Schema SSOT (SQLite): tables, columns, PK, FK, indexes |
| Legacy Java codebase | `code-ast-ssot` (planned) | Code SSOT: compilation units, types, methods, annotations, references |

Both exporters are **read-only** on source systems and produce **versioned snapshots** suitable for drift comparison.

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

## Data contracts

See [SSOT-SCHEMA.md](SSOT-SCHEMA.md) for cross-repo schema versioning and entity key conventions.

### Schema SSOT (implemented)

- Repository: `db-metadata`
- Format: SQLite v1 (`export_run`, `db_table`, `db_column`, `db_foreign_key`, …)
- Stable IDs: `schema.table.column`

### Code AST SSOT (planned)

- Repository: `code-ast-ssot`
- Format: TBD (SQLite or graph export)
- Stable IDs: TBD (e.g. `module.package.Type#method`)

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
- `demo-legacy-app` sample project for end-to-end demos
- Org-level GitHub Actions for cross-repo smoke tests
