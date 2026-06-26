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

Planned folders (may not exist yet): `demo-dukesbank/`, `code-ast-ssot/`.

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
| Duke's Bank | Reference app; EJB 2.x CMP + XML descriptors — DRG is Java **and** XML |
| Code SSOT parser | **JavaParser + XML**, not OpenRewrite LST for long-term SSOT |
| LST | Use only for OpenRewrite recipe development (`rewrite-recipes`, future) |
| Comments | Store separately; no v1 semantic comment→statement mapping |
| Repos | Multi-repo under org; not a monorepo |

See lab-notes ADR-001 (private) or public DUKESBANK-DEMO § AST vs LST.

## Current status (update via journal if stale)

- **Done:** `db-metadata` alpha, `verify`, CI-oriented tests, migration-hub docs, GitHub org, lab-notes private repo  
- **Not done:** `demo-dukesbank` Docker, Duke's Bank MySQL export POC, `code-ast-ssot`

## Typical tasks

| User ask | Where to work |
|----------|----------------|
| Export / verify database schema | `db-metadata/` |
| Docker Duke's Bank MySQL | create `demo-dukesbank/` under workspace root |
| Java AST / EJB XML extraction | future `code-ast-ssot/` — read DUKESBANK-DEMO Phase B |
| Program docs / blog outline | `migration-hub/docs/` or private `lab-notes/blog-drafts/` |
| Session log | private `lab-notes/journal/` |

## Workspace file

User opens `anchor-migration.code-workspace` for multi-root: migration-hub, db-metadata, lab-notes.

Root pointer: `../README.md` (parent of migration-hub — local workspace root).
