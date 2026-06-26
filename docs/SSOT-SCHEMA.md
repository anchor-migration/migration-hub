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
**Design:** [ADR-002 — core vs stack profiles](ADR-002-java-ast-ssot-core-and-profiles.md), [ADR-003 — sidecars vs LST](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md)  
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

### Future profiles (planned)

| Profile | Inputs | Purpose |
|---------|--------|---------|
| `spring` | `@Configuration`, component scan, XML | Spring bean graph |
| `jpa` | `@Entity`, `persistence.xml` | JPA ↔ schema crosswalk |

Each profile adds tables or extension rows; core IDs remain stable.

### Sidecars (planned)

Optional layers on core AST — not full OpenRewrite LST. See [ADR-003](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md).

| Sidecar | Table (planned) | Purpose |
|---------|-----------------|---------|
| Comments | `source_comment` | Raw comment blocks; no v1 semantic comment→statement edges |
| Source span | `source_span` (idea) | Node stable ID ↔ file line range |

## Linking schema SSOT ↔ Java AST SSOT

Future crosswalk table (location TBD):

| Code entity | Schema entity | Link type |
|-------------|---------------|-----------|
| `@Entity` class (JPA profile) | `db_table` row | `maps_to_table` |
| EJB entity bean (javaee profile) | `db_table` row | `ejb_to_table` → schema |
| `@Column` field | `db_column` row | `maps_to_column` |
| `@JoinColumn` | `db_foreign_key` row | `maps_to_fk` |

Both sides must reference the same `export_run_id` (or compatible snapshot timestamps).

## OpenRewrite recipe inputs (planned)

Recipes parse source to **LST at apply time** (not from SSOT storage). They may also read:

- Schema SSOT: table/column names, FK graph for relationship migrations
- Java AST SSOT (core): type hierarchy for recipe targeting
- Profile crosswalks (e.g. EJB/XML, JPA): entity ↔ table binding

Contract: recipe modules declare required SSOT versions in `recipe.yml` metadata.

## Parity verification inputs (planned)

Parity verifier consumes:

- Old and new AST snapshots (or live parse)
- Optional: schema SSOT for data fixture generation
- Pattern catalog entry ID for expected behavioral scope

Output: machine-readable parity report with pass / fail / unknown per check.

## Versioning policy

1. Increment SSOT major version on breaking DDL or ID format changes.
2. Exporters write `meta_schema_version` (schema SSOT) or equivalent.
3. Downstream tools reject unknown major versions with a clear error.
4. Document migration notes between SSOT versions in each repo's CHANGELOG.
