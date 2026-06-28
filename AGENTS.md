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
| `anchor-explorer/` | public | Read-only UI: load linked SQLite, crosswalk graph + colors |

## Hard conventions

- User may chat in **Chinese**; **code comments, documentation, commit messages** → **English**.
- Do not commit secrets (`.env`, credentials).
- Only commit when the user explicitly asks.
- Do not edit attached plan files unless asked.
- Public docs must not link to or expose private lab-notes content.
- On [ADR-006](docs/ADR-006-multi-role-decision-review.md) **gate triggers** (new repo, boundary protocol change, new profile/edge_kind, user-visible semantics): complete the decision review template and lock ADR + acceptance criteria **before** bulk implementation — do not skip to coding.

## Key technical decisions (already made)

| Topic | Decision |
|-------|----------|
| Schema SSOT | SQLite snapshots via `db-metadata`; `export_run` versioning; `verify` reconciles against live DB |
| Duke's Bank | **Reference demo** only; validates profile `javaee-ejb2-jboss`, not `java-ast-ssot` product scope |
| `java-ast-ssot` | **Core:** JavaParser AST. **Profiles:** stack adapters (EJB/XML today). See [ADR-002](docs/ADR-002-java-ast-ssot-core-and-profiles.md) |
| Code SSOT parser | **JavaParser** for core; deployment XML via **profiles** — not OpenRewrite LST for SSOT |
| LST | Transform-time only in `rewrite-recipes`; not stored as SSOT — see [ADR-003](docs/ADR-003-ast-sidecar-vs-lst-rewrite-layer.md) |
| Rewrite phasing | Session→`@Service` via **BeanState** (3.2), then scalar CMP→JPA `AccountBean` (3.3) — [ADR-007](docs/ADR-007-rewrite-recipes-session-and-cmp-jpa.md) |
| Language modernization | L1/L2/L3; **`classify-lists`** (M2, on-demand JSON) before L2; tuple `List` → result class — [ADR-008](docs/ADR-008-java-language-modernization-and-tuple-lists.md) |
| Comments | Optional `source_comment` sidecar; no v1 semantic comment→statement mapping — [ADR-003](docs/ADR-003-ast-sidecar-vs-lst-rewrite-layer.md) |
| Crosswalk | Profile extract + link; mapping tiers; **edge colors** (green/yellow/orange/red) — [ADR-004](docs/ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md), [ADR-005](docs/ADR-005-multi-tier-alignment-and-ssot-explorer.md) |
| Anchor Explorer | Read-only human UI over SQLite snapshots — first-class interface — [ADR-005](docs/ADR-005-multi-tier-alignment-and-ssot-explorer.md) |
| Decision process | Multi-role review before gated implementation — [ADR-006](docs/ADR-006-multi-role-decision-review.md) |
| AST repo naming | **`{language}-ast-ssot`** (e.g. `java-ast-ssot`); reserved for future e.g. `cobol-ast-ssot` |
| Repos | Multi-repo under org; not a monorepo |

See [ADR-003](docs/ADR-003-ast-sidecar-vs-lst-rewrite-layer.md), lab-notes ADR-001 (private), or DUKESBANK-DEMO § AST vs LST.

## Current status (update via journal if stale)

- **Done:** `db-metadata` alpha; Duke's Bank E2E; `java-ast-ssot` v1.0 (core + profiles + crosswalk + **`classify-lists` M2**); `anchor-explorer` alpha; `rewrite-recipes` 3.0–3.3 + ADR-008 L1 + ADR-009 presets; ADR-002–009  
- **Next:** ADR-008 M3 L2 homogeneous recipe (`rewrite-recipes`); `jpa` profile ([ADR-004](docs/ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md))

## Typical tasks

| User ask | Where to work |
|----------|----------------|
| Export / verify database schema | `db-metadata/` |
| Docker Duke's Bank MySQL | `demo-dukesbank/` |
| Java AST / EJB XML extraction | `java-ast-ssot/` — core + `--profile javaee-ejb2-jboss`; read ADR-002 + ADR-004 + DUKESBANK-DEMO Phase B |
| Raw list usage (homogeneous / tuple) | `java-ast-ssot classify-lists` — on-demand JSON, no cache; [list-usage-classifier.md](https://github.com/anchor-migration/java-ast-ssot/blob/main/docs/list-usage-classifier.md) |
| Code ↔ schema crosswalk | `java-ast-ssot crosswalk` — `--code-db`, `--schema-db`, `--db-schema`, `-o`; see ADR-004 |
| Program docs / blog outline | `migration-hub/docs/` or private `lab-notes/blog-drafts/` |
| Session log | private `lab-notes/journal/` |

## Workspace file

User opens `anchor-migration.code-workspace` for multi-root: migration-hub, db-metadata, java-ast-ssot, demo-dukesbank, lab-notes.

Root pointer: `../README.md` (parent of migration-hub — local workspace root).
