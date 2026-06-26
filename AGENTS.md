# AGENTS.md — Anchor Migration

Instructions for AI coding assistants (Cursor, etc.) in a **new session** with no prior chat history.

## Bootstrap (read first)

1. [docs/START-HERE.md](docs/START-HERE.md) — program map, repos, conventions  
2. If user mentions recent work: private repo `anchor-migration-lab-notes` → latest `journal/*-session-wrapup.md`  
3. Task-specific:
   - Schema / DB → [db-metadata README](https://github.com/anchor-migration/db-metadata)
   - Duke's Bank / Java DRG → [docs/DUKESBANK-DEMO.md](docs/DUKESBANK-DEMO.md)
   - How we build → [docs/DEVELOPMENT-MODEL.md](docs/DEVELOPMENT-MODEL.md)

**Do not assume** you remember previous conversations. Use files in the workspace.

## Program summary

- **Goal:** SSOT-driven legacy Java migration (EJB-era samples like Duke's Bank → modern stack).
- **Public org:** https://github.com/anchor-migration
- **Deterministic deliverables:** Python (`db-metadata`) and future Java tools — reproducible, tested.
- **AI role:** implement under architecture and boundary protocols defined by the developer; not a replacement for SSOT or verification.

## Repository layout

| Path | Repo | Notes |
|------|------|-------|
| `migration-hub/` | public | Docs only — START-HERE, ARCHITECTURE, ROADMAP |
| `db-metadata/` | public | Python CLI: `db-migration export|verify|info` |
| `lab-notes/` | **private** | Journals, ADR drafts, blog drafts — may contain WIP |
| `../dukesbank/` or `C:\github\dukesbank` | external | Duke's Bank J2EE 1.4 sample, MySQL SQL in `data/mysql/` |
| `demo-dukesbank/` | public | MySQL Docker bridge for Duke's Bank |
| `java-ast-ssot/` | public | Java AST SSOT CLI (core + stack profiles) |

## Hard conventions

- User may chat in **Chinese**; **code comments, documentation, commit messages** → **English**.
- Do not commit secrets (`.env`, credentials).
- Only commit when the user explicitly asks.
- Do not edit attached plan files unless asked.
- Public docs must not link to or expose private lab-notes content.

## Key technical decisions (already made)

| Topic | Decision |
|-------|----------|
| Schema SSOT | SQLite snapshots via `db-metadata`; `export_run` versioning; `verify` reconciles against live DB |
| Duke's Bank | **Reference demo** only; validates profile `javaee-ejb2-jboss`, not `java-ast-ssot` product scope |
| `java-ast-ssot` | **Core:** JavaParser AST. **Profiles:** stack adapters (EJB/XML today). See [ADR-002](docs/ADR-002-java-ast-ssot-core-and-profiles.md) |
| Code SSOT parser | **JavaParser** for core; deployment XML via **profiles** — not OpenRewrite LST for SSOT |
| LST | Use only for OpenRewrite recipe development (`rewrite-recipes`, future) |
| Comments | Store separately; no v1 semantic comment→statement mapping |
| AST repo naming | **`{language}-ast-ssot`** (e.g. `java-ast-ssot`); reserved for future e.g. `cobol-ast-ssot` |
| Repos | Multi-repo under org; not a monorepo |

See lab-notes ADR-001 (private) or public DUKESBANK-DEMO § AST vs LST.

## Current status (update via journal if stale)

- **Done:** `db-metadata` alpha, Duke's Bank MySQL demo, `java-ast-ssot` alpha (monolithic v0.1), ADR-002 docs  
- **Next:** ADR-002 Step 2 refactor (`--profile`, core-only export), crosswalk CLI, `rewrite-recipes`

## Typical tasks

| User ask | Where to work |
|----------|----------------|
| Export / verify database schema | `db-metadata/` |
| Docker Duke's Bank MySQL | `demo-dukesbank/` |
| Java AST / EJB XML extraction | `java-ast-ssot/` — core + `--profile javaee-ejb2-jboss`; read ADR-002 + DUKESBANK-DEMO Phase B |
| Program docs / blog outline | `migration-hub/docs/` or private `lab-notes/blog-drafts/` |
| Session log | private `lab-notes/journal/` |

## Workspace file

User opens `anchor-migration.code-workspace` for multi-root: migration-hub, db-metadata, java-ast-ssot, demo-dukesbank, lab-notes.

Root pointer: `../README.md` (parent of migration-hub — local workspace root).
