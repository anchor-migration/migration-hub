# Roadmap

Status legend: ✅ Done · 🚧 In progress · 📋 Planned · 💡 Idea

## Phase 0 — Foundation

| Item | Repo | Status |
|------|------|--------|
| Program hub and architecture docs | migration-hub | ✅ |
| [ADR-006](docs/ADR-006-multi-role-decision-review.md) — multi-role decision gate | migration-hub | ✅ |
| Schema export CLI (multi-DB) | db-metadata | ✅ Alpha |
| Export verification (source vs SQLite) | db-metadata | ✅ Alpha |
| Local workspace layout | anchor-migration/ | ✅ |
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
| Profile `jpa` — `@Entity` / `@Table` / `@Column` crosswalk | java-ast-ssot | ✅ Alpha (ADR-004 Step 4) |
| Profile `mybatis` — mapper XML + JOIN read models | java-ast-ssot | ✅ Alpha (ADR-004 Step 5) |
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
| Scalar CMP→JPA — `AccountBean` only (3.3) | rewrite-recipes | ✅ |
| v0.4a `CmpManyToManyToJpa` — `AccountBean.customers` | rewrite-recipes | ✅ |
| v0.4b `CmpForeignKeyToJpa` — `TxBean.account` | rewrite-recipes | ✅ |
| v0.4c `CmpScalarEntityToJpa` — `CustomerBean`, `TxBean` | rewrite-recipes | ✅ |
| v0.4d `NextIdTableToJpa` — `NextIdBean` (retains `getNextId()`) | rewrite-recipes | ✅ |
| Duke's Bank multi-entity JPA E2E + parity gates | demo-dukesbank + parity-verify | ✅ |
| EJB `@Stateless` → `@Service` annotation-only | rewrite-recipes | ❌ Rejected (ADR-007) |
| [ADR-008](docs/ADR-008-java-language-modernization-and-tuple-lists.md) — L1/L2/L3 language modernization | migration-hub | ✅ |
| L1: `Vector` → `ArrayList` (+ L1 YAML composite) | rewrite-recipes | ✅ |
| [ADR-009](docs/ADR-009-rewrite-engine-presets-and-run-manifest.md) — preset manifests | migration-hub + rewrite-recipes | ✅ |
| Preset catalog (`Smoke`, `LanguageL1Only`, `LanguageL2Only`, `LanguageL3Only`, `DukesBankStackMigration`) | rewrite-recipes | ✅ |
| M2: tuple vs homogeneous list classifier (SSOT) | java-ast-ssot | ✅ (on-demand CLI, no cache) |
| L2: homogeneous raw `ArrayList` typing | rewrite-recipes | ✅ |
| L3: tuple list → result class (proposal + human review) | rewrite-recipes | ✅ |

## Phase 4 — Parity verification

| Item | Repo | Status |
|------|------|--------|
| AST subtree diff (old vs new) | parity-verify | ✅ Beta (v0.2 JSON + HTML) |
| Behavioral matrix (`dukesbank-cmp-jpa`) | parity-verify | ✅ Beta (YAML-backed) |
| Multi-entity parity matrices (`dukesbank-cmp-jpa-multi-*`) | parity-verify | ✅ Beta |
| Duke's Bank JPA E2E (`run-e2e-jpa-parity.ps1`) | demo-dukesbank | ✅ |
| Custom matrix YAML loader (`--matrix-file`) | parity-verify | ✅ Beta |
| Pattern-catalog checklists (`pattern_id`) | parity-verify + pattern-catalog | ✅ Alpha |
| AI-assisted test stub generation (`generate-tests`) | parity-verify | ✅ Beta |
| Parity report format (HTML / JSON) | parity-verify | ✅ JSON + HTML |

## Phase 5 — Ecosystem

| Item | Repo | Status |
|------|------|--------|
| [ADR-010](docs/ADR-010-stubborn-integration.md) — stubborn horizontal LLM context | migration-hub + stubborn | ✅ |
| `stubborn` v0.3 — token budget, `metrics` KPI, Docker E2E | stubborn | ✅ (superseded by Beta) |
| `stubborn` Beta (`0.10.0b2`) — weave switches, Duke's Bank runbook | stubborn + demo-dukesbank | ✅ |
| `stubborn` weave switches (`member-signatures`, `javadoc`) | stubborn | ✅ Beta |
| `stubborn` MCP server (`stubborn mcp`) | stubborn | ✅ Beta (see `docs/MCP.md`) |
| migration-bridge example (Duke's Bank LLM workflow) | stubborn | ✅ |
| Pattern catalog (6 documented CMP→JPA patterns) | pattern-catalog | ✅ Alpha |
| Demo legacy application | demo-legacy-app | 💡 |
| Contributing guide and good-first issues | all | 📋 |
| End-to-end video / blog walkthrough | migration-hub | 💡 |

## Non-goals (for now)

- Full autonomous migration without human review
- Supporting every legacy framework in v1
- Monorepo consolidation (repos stay separate)

## How to contribute

Pick a 📋 item, open an issue in the target repo, and reference this roadmap. Schema SSOT (`db-metadata`) is the best entry point today.
