# SSOT Schema Contracts

Cross-repository contracts for Anchor Migration artifacts. Version bumps must be documented here and in each exporter's DDL.

## Schema SSOT (database metadata)

**Owner:** `db-metadata`  
**Version:** 1  
**Format:** SQLite  
**DDL:** [db-metadata/src/db_migration/store/schema/v1.sql](https://github.com/anchor-migration/db-metadata/blob/main/src/db_migration/store/schema/v1.sql)

### Snapshot boundary

Each export creates one `export_run` row. Downstream tools should pin `export_run_id` or explicitly use the latest run.

### Stable identifiers

| Entity | Format | Example |
|--------|--------|---------|
| Table | `{schema}.{table}` | `public.orders` |
| Column | `{schema}.{table}.{column}` | `public.orders.id` |
| Primary key | `{schema}.{table}\|{column}\|{ordinal}` | `public.orders\|id\|1` |
| Foreign key | `{src_schema}.{src_table}\|{src_col}\|{tgt_schema}.{tgt_table}\|{tgt_col}` | `public.orders\|customer_id\|public.customers\|id` |
| Index | `{schema}.{table}\|{index_name}` | `public.orders\|idx_orders_customer` |

Verification scripts in `db-metadata` use the same keys for reconciliation.

### Consumption guidelines

- **read-only** — downstream tools must not mutate SSOT files; create derived artifacts instead
- **pin snapshots** — record `export_run_id` and `tool_version` in migration logs
- **schema drift** — re-export and diff `export_run` rows before major refactors

## Java AST SSOT

**Owner:** `java-ast-ssot`  
**Design:** [ADR-002 — core vs stack profiles](ADR-002-java-ast-ssot-core-and-profiles.md), [ADR-003 — sidecars vs LST](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md), [ADR-004 — crosswalk contract](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md)  
**Format:** SQLite  

Language-specific by design — not a generic `code-ast-ssot`. The repo exports **Java source structure**; legacy stack bindings (Java EE, Spring, …) are **optional profiles**.

### Core (always)

**DDL (v1, mixed file today):** [java-ast-ssot/.../schema/v1.sql](https://github.com/anchor-migration/java-ast-ssot/blob/main/src/main/resources/schema/v1.sql) — refactor will isolate core tables.

| Entity | Stable ID |
|--------|-----------|
| Type | `{package}.{SimpleName}` (nested: `{outer}$.{Inner}`) |
| Method | `{Type}#{methodName}({paramTypes})` |
| Field | `{Type}#{fieldName}` |

Tables: `source_file`, `java_type`, `java_method`, `java_field`, `java_import`.

### Profile: `javaee-ejb2-jboss` (v0.1)

Enabled when EJB 2.x + JBoss CMP descriptors are present (today: implicit; future: `--profile javaee-ejb2-jboss`).

| Entity | Stable ID |
|--------|-----------|
| EJB | `ejb:{ejbName}` |
| Table (crosswalk target) | `{schema}.{TABLE}` |

**DDL (core):** `schema/core/v1.sql` in repo  
**DDL (profile):** `schema/profiles/javaee-ejb2-jboss/v1.sql`

Tables: `javaee_ejb2_jboss_bean`, `javaee_ejb2_jboss_cmp_field`, `javaee_ejb2_jboss_ref`, `javaee_ejb2_jboss_crosswalk`.

**Reference validation:** Duke's Bank bank module — not a product boundary.

#### Profile `jpa` (implemented — scalar v1)

Tables: `jpa_entity`, `jpa_field`. Extracts `@Entity`, `@Table`, `@Column`, `@Id` from `.java` sources. Crosswalk contributor emits `type_maps_to_table` and `field_maps_to_column` with `binding_source` = `jpa_annotation` (no EJB-style `stack_bridge`).

#### Profile `mybatis` (implemented — v1)

Tables: `mybatis_result_map`, `mybatis_result_field`, `mybatis_statement`, `mybatis_statement_table`. Parses `*Mapper.xml` `resultMap` bindings and classifies single-table vs JOIN `select` statements. Crosswalk emits `persistent_entity` or `read_model` edges per ADR-004 (`type_backed_by_sql`, `sql_references_table`, `field_maps_to_column_via` for JOIN DTOs).

### Future profiles (planned)

| Profile | Inputs | Purpose |
|---------|--------|---------|
| `spring` | `@Configuration`, component scan, XML | Spring bean graph |

Each profile adds tables or extension rows; core IDs remain stable.

### Sidecars (planned)

Optional layers on core AST — not full OpenRewrite LST. See [ADR-003](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md).

| Sidecar | Table | Purpose |
|---------|-------|---------|
| Comments | `source_comment` | Raw comment blocks; no v1 semantic comment→statement edges — **implemented** in `java-ast-ssot` core export |
| Source span | `source_span` (idea) | Node stable ID ↔ file line range |

## Linking schema SSOT ↔ Java AST SSOT

**Design:** [ADR-004 — crosswalk contract](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md)

Crosswalk is **two steps**: (1) stack **profiles** extract binding facts during code export; (2) **`java-ast-ssot crosswalk`** normalizes facts into canonical edges and validates them against schema SSOT.

### Mapping roles

Not every Java type maps 1:1 to a table. Each linked type carries a **`mapping_role`**:

| Role | Schema link pattern |
|------|---------------------|
| `persistent_entity` | `type_maps_to_table` + `field_maps_to_column` |
| `read_model` | `type_backed_by_sql` + `field_maps_to_column_via` + `sql_references_table` |
| `transfer_object` | Usually no table edges |
| `repository_boundary` | `method_executes_sql` → SQL artifact |

See ADR-004 for parity policy per role.

### Canonical edge kinds (`code_schema_link`)

| edge_kind | Code side | Schema / SQL side |
|-----------|-----------|-------------------|
| `type_maps_to_table` | `{package}.{Type}` | `{schema}.{table}` |
| `field_maps_to_column` | `{Type}#{field}` | `{schema}.{table}.{column}` |
| `field_maps_to_column_via` | `{Type}#{field}` | `{schema}.{table}.{column}` (JOIN/alias) |
| `type_backed_by_sql` | `{package}.{Type}` | `sql:{mapper}#{statementId}` |
| `method_executes_sql` | `{Type}#{method}(…)` | `sql:{mapper}#{statementId}` |
| `sql_references_table` | `sql:…` | `{schema}.{table}` |
| `relationship_maps_to_table` | `ejb:{name}` or `{Type}` | xref / link table |
| `stack_bridge` | Profile-specific intermediate | Profile-specific |

Metadata per row: `profile_id`, `binding_source`, `evidence_ref`, `confidence` (`authoritative` \| `inferred`), `code_export_run_id`, `schema_export_run_id`.

**Alignment quality (planned — [ADR-005](ADR-005-multi-tier-alignment-and-ssot-explorer.md)):** `name_drift_class`, `type_relation_forward` / `type_relation_backward`, **`color_forward` / `color_backward`** (bidirectional traffic-map coloring), optional `round_trip_class`, `mapping_tier`. Full match → green both ways; e.g. `int`↔`long` → green forward, yellow backward.

### Profile binding signals (summary)

| Profile | Primary signals | Typical role |
|---------|-----------------|--------------|
| `javaee-ejb2-jboss` | `ejb-jar.xml`, `jbosscmp-jdbc.xml` | `persistent_entity` |
| `jpa` | `@Entity`, `@Table`, `@Column`, `@Id` | `persistent_entity` (`type_maps_to_table`, `field_maps_to_column`) |
| `mybatis` | mapper XML (`resultMap`, JOIN `select`) | `persistent_entity` or `read_model` |

Profile-local edges (e.g. `ejb_to_table`) are normalized at link time — see ADR-004.

### Planned link CLI

```bash
java-ast-ssot crosswalk \
  --code-db metadata/dukesbank-code.db \
  --schema-db metadata/dukesbank.db \
  --db-schema dukesbank \
  --out metadata/dukesbank-linked.db
```

Both input snapshots must pin compatible `export_run_id` values (or explicit run flags). Link output is a third SQLite file in v1.

## OpenRewrite recipe inputs

Recipes parse source to **LST at apply time** (not from SSOT storage). They may also read:

- Schema SSOT: table/column names, FK graph for relationship migrations
- Java AST SSOT (core): type hierarchy for recipe targeting
- Profile crosswalks (e.g. EJB/XML, JPA): entity ↔ table binding

Contract: recipe modules declare required SSOT versions in `recipe.yml` metadata. Implemented in [rewrite-recipes](https://github.com/anchor-migration/rewrite-recipes) (ADR-007 stack + ADR-008 language modernization).

## Parity verification inputs

**Owner:** `parity-verify` (Beta v0.2)

Parity verifier consumes:

- Old and new AST snapshots (`java-ast-ssot` exports)
- Optional: linked crosswalk DBs (`--linked-before`, `--linked-after`)
- Behavioral matrix YAML (`--matrix` / `--matrix-file`); optional `pattern_id` via [pattern-catalog](https://github.com/anchor-migration/pattern-catalog)
- Optional: `--touchpoint-source` for migrated source file checks

Output: JSON + HTML parity report with pass / fail / unknown per structural and behavioral check. CLI: `parity-verify compare`; see [parity-verify README](https://github.com/anchor-migration/parity-verify).

## Symbol graph SSOT (LLM context)

**Owner:** `anchor-stubborn`  
**Design:** [ADR-010 — anchor-stubborn integration](ADR-010-anchor-stubborn-integration.md)  
**Version:** 1  
**Format:** SQLite  
**DDL:** [anchor-stubborn/src/anchor_stubborn/store/schema/v1.sql](https://github.com/stubborn-ai/stubborn/blob/main/src/anchor_stubborn/store/schema/v1.sql)

Separate from Java AST SSOT. Ingests SCIP symbol indexes; stores symbols, references, and document metadata for pruning and weaving.

### Consumption guidelines

- **read-only** — treat like other SSOT snapshots; derive stub text or metrics, do not mutate in place
- **optional** — migration runbooks do not require this artifact for export, crosswalk, rewrite, or parity
- **privacy** — woven output excludes method bodies; see ADR-010 privacy contract

## Versioning policy

1. Increment SSOT major version on breaking DDL or ID format changes.
2. Exporters write `meta_schema_version` (schema SSOT) or equivalent.
3. Downstream tools reject unknown major versions with a clear error.
4. Document migration notes between SSOT versions in each repo's CHANGELOG.
