# Roadmap

Status legend: ✅ Done · 🚧 In progress · 📋 Planned · 💡 Idea

## Phase 0 — Foundation

| Item | Repo | Status |
|------|------|--------|
| Program hub and architecture docs | migration-hub | 🚧 |
| Schema export CLI (multi-DB) | db-metadata | ✅ Alpha |
| Export verification (source vs SQLite) | db-metadata | ✅ Alpha |
| Local workspace layout | anchor-migration/ | 🚧 |
| GitHub Organization setup | anchor-migration | 📋 |

## Phase 1 — Schema SSOT hardening

| Item | Repo | Status |
|------|------|--------|
| [Duke's Bank demo doc](docs/DUKESBANK-DEMO.md) — design + runbook | migration-hub | ✅ |
| Docker compose for Duke's Bank MySQL | demo-dukesbank | ✅ |
| Real-DB verification on Duke's Bank MySQL | db-metadata | ✅ |
| `export_run` diff subcommand | db-metadata | 💡 |
| View / procedure definition extraction (v2 schema) | db-metadata | 💡 |

## Phase 2 — Java AST SSOT

| Item | Repo | Status |
|------|------|--------|
| [ADR-002](docs/ADR-002-java-ast-ssot-core-and-profiles.md) — core vs profiles | migration-hub | ✅ |
| [Duke's Bank DRG design](DUKESBANK-DEMO.md) — AST/XML/LST decisions | migration-hub | ✅ |
| JavaParser prototype (core) | java-ast-ssot | ✅ Alpha |
| Profile `javaee-ejb2-jboss` on Duke's Bank | java-ast-ssot | ✅ Alpha (implicit in v0.1) |
| **Refactor:** `--profile`, core-only export, schema split | java-ast-ssot | 📋 Step 2 |
| Cross-reference: profile crosswalk ↔ schema SSOT | java-ast-ssot + db-metadata | 📋 |

## Phase 3 — OpenRewrite integration

| Item | Repo | Status |
|------|------|--------|
| Recipe template and testing harness | rewrite-recipes | 📋 |
| EJB `@Stateless` → Spring `@Service` recipe | rewrite-recipes | 📋 |
| Pattern: JPA entity alignment from schema SSOT | rewrite-recipes | 📋 |

## Phase 4 — Parity verification

| Item | Repo | Status |
|------|------|--------|
| AST subtree diff (old vs new) | parity-verify | 📋 |
| AI-assisted test case generation (bounded) | parity-verify | 📋 |
| Parity report format (HTML / JSON) | parity-verify | 📋 |

## Phase 5 — Ecosystem

| Item | Repo | Status |
|------|------|--------|
| Pattern catalog (5+ documented patterns) | pattern-catalog | 📋 |
| Demo legacy application | demo-legacy-app | 💡 |
| Contributing guide and good-first issues | all | 📋 |
| End-to-end video / blog walkthrough | migration-hub | 💡 |

## Non-goals (for now)

- Full autonomous migration without human review
- Supporting every legacy framework in v1
- Monorepo consolidation (repos stay separate)

## How to contribute

Pick a 📋 item, open an issue in the target repo, and reference this roadmap. Schema SSOT (`db-metadata`) is the best entry point today.
