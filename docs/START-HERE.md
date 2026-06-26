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
| 2 | [ARCHITECTURE.md](ARCHITECTURE.md) | Layers: extract → transform → verify |
| 3 | [SSOT-SCHEMA.md](SSOT-SCHEMA.md) | Cross-repo contracts, stable IDs |
| 4 | [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md) | Reference app, DRG design, AST/LST/XML decisions |
| 5 | [ROADMAP.md](ROADMAP.md) | What is done vs planned |
| 6 | [db-metadata README](https://github.com/anchor-migration/db-metadata) | CLI: `export`, `verify`, `info` |

**Private (if you have access):** `lab-notes/journal/2026-06-27-session-wrapup.md` — latest session log.

## Repository map

| Repo | Visibility | Role | Status |
|------|------------|------|--------|
| **migration-hub** | public | Program docs (this repo) | Active |
| **db-metadata** | public | Live DB → schema SSOT (SQLite) | Alpha |
| **.github** | public | Org profile README | Active |
| **lab-notes** | **private** | Journals, ADR drafts, blog drafts | Active |
| **java-ast-ssot** | public | Java/XML → Java AST SSOT | Alpha |
| **rewrite-recipes** | public | OpenRewrite catalog | Planned |
| **parity-verify** | public | Old vs new parity | Planned |
| **pattern-catalog** | public | Migration patterns | Planned |

## Local workspace (author machine)

```
C:\github\anchor-migration\
├── migration-hub/
├── db-metadata/
├── lab-notes/              → github.com/anchor-migration/anchor-migration-lab-notes (private)
├── demo-dukesbank/         MySQL bridge for Duke's Bank demo
├── java-ast-ssot/          Java AST SSOT exporter
└── anchor-migration.code-workspace

C:\github\dukesbank\        Duke's Bank legacy sample (external to org)
```

## Pipeline (one picture)

```
Live MySQL ──db-metadata──► schema SSOT (SQLite)
Legacy Java + XML ──java-ast-ssot──► Java AST SSOT (SQLite)
         └─ crosswalk ─► entity ↔ table links
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

## Language-specific AST repos

Extractors use **`{language}-ast-ssot`** names (e.g. `java-ast-ssot`). This leaves room for future heterogeneous migration repos (e.g. `cobol-ast-ssot`) without overloading one generic `code-ast-ssot`.

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

1. Publish `demo-dukesbank` and `java-ast-ssot` to GitHub org
2. `java-ast-ssot crosswalk` — link Java/EJB SSOT to `db-metadata` schema SSOT
3. Blog draft from [lab-notes backlog](https://github.com/anchor-migration/anchor-migration-lab-notes) (private)

## For AI assistants

See **[AGENTS.md](../AGENTS.md)** at the root of this repository for session bootstrap instructions.

When the user says “continue Anchor Migration”, read in order:

1. This file  
2. `AGENTS.md`  
3. Latest `lab-notes/journal/*-session-wrapup.md` (if available)  
4. `DUKESBANK-DEMO.md` if work touches Duke's Bank or code SSOT  

Do **not** assume chat history from prior sessions exists.
