# Roadmap

Status legend: ✅ Done · 🚧 In progress · 📋 Planned · 💡 Idea

## Phase 0 — Foundation

| Item | Repo | Status |
|------|------|--------|
| Program hub and architecture docs | migration-hub | 🚧 |
| [ADR-006](docs/ADR-006-multi-role-decision-review.md) — multi-role decision gate | migration-hub | ✅ |
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
| [ADR-003](docs/ADR-003-ast-sidecar-vs-lst-rewrite-layer.md) — AST sidecars vs LST rewrite layer | migration-hub | ✅ |
| [ADR-004](docs/ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md) — crosswalk mapping roles & edge kinds | migration-hub | ✅ |
| [Duke's Bank DRG design](DUKESBANK-DEMO.md) — AST/XML/LST decisions | migration-hub | ✅ |
| JavaParser prototype (core) | java-ast-ssot | ✅ Alpha |
| Profile `javaee-ejb2-jboss` on Duke's Bank | java-ast-ssot | ✅ Alpha (implicit in v0.1) |
| **Refactor:** `--profile`, core-only export, schema split | java-ast-ssot | ✅ 1.0 (breaking) |
| Cross-reference: profile crosswalk ↔ schema SSOT (`crosswalk` CLI) | java-ast-ssot + db-metadata | ✅ Alpha — see [ADR-004](docs/ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md) |

## Phase 2.5 — SSOT Explorer (human interface)

| Item | Repo | Status |
|------|------|--------|
| [ADR-005](docs/ADR-005-multi-tier-alignment-and-ssot-explorer.md) — tiers, drift classes, edge coloring | migration-hub | ✅ |
| **`anchor-explorer`** — crosswalk graph + color legend (Duke's Bank) | anchor-explorer | ✅ Alpha (E2E verified) |
| Duke's Bank E2E runbook (crosswalk + Explorer) | migration-hub + demo-dukesbank | ✅ |
| Link metadata: `name_drift_class`, `type_relation`, bidirectional colors | java-ast-ssot | ✅ Alpha |
| Domain / presentation tier edges | java-ast-ssot profiles | 💡 |

## Phase 3 — OpenRewrite integration

> Scope and phasing: [ADR-007](docs/ADR-007-rewrite-recipes-session-and-cmp-jpa.md) (Accepted).

| Item | Repo | Status |
|------|------|--------|
| [ADR-007](docs/ADR-007-rewrite-recipes-session-and-cmp-jpa.md) — Session→Service (BeanState) vs CMP→JPA | migration-hub | ✅ |
| Recipe template and testing harness (3.0) | rewrite-recipes | ✅ |
| CMP→JPA capability matrix — Duke's Bank (3.1a) | rewrite-recipes | ✅ |
| Session `BeanState` spike — `AccountControllerBean` (3.1b) | rewrite-recipes | ✅ |
| Session→Service recipe chain — `AccountControllerBean` (3.2) | rewrite-recipes | ✅ |
| Scalar CMP→JPA — `AccountBean` only (3.3) | rewrite-recipes | 📋 |
| EJB `@Stateless` → `@Service` annotation-only | rewrite-recipes | ❌ Rejected (ADR-007) |
| [ADR-008](docs/ADR-008-java-language-modernization-and-tuple-lists.md) — L1/L2/L3 language modernization | migration-hub | ✅ |
| L1: `Vector` → `ArrayList` | rewrite-recipes | 📋 |
| M2: tuple vs homogeneous list classifier (SSOT) | java-ast-ssot / rewrite-recipes | 📋 |
| L3: tuple list → result class (proposal + human review) | rewrite-recipes | 📋 |

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
