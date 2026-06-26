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
**Version:** 1  
**Format:** SQLite  
**DDL:** [java-ast-ssot/src/main/resources/schema/v1.sql](https://github.com/anchor-migration/java-ast-ssot/blob/main/src/main/resources/schema/v1.sql)

Language-specific by design — not a generic `code-ast-ssot`. Future repos (e.g. `cobol-ast-ssot`) may follow the same contract pattern for heterogeneous migration.

### Entity types (v1)

- source file, Java type, method, field, import
- EJB bean, CMP field, EJB ref (from deployment XML)
- crosswalk edge (`java_type_to_ejb`, `ejb_to_table`)

### Stable IDs (v1)

| Entity | Format |
|--------|--------|
| Type | `{package}.{SimpleName}` (nested: `{outer}$.{Inner}`) |
| Method | `{Type}#{methodName}({paramTypes})` |
| Field | `{Type}#{fieldName}` |
| EJB | `ejb:{ejbName}` |
| Table (crosswalk target) | `{schema}.{TABLE}` |

## Linking schema SSOT ↔ Java AST SSOT

Future crosswalk table (location TBD):

| Code entity | Schema entity | Link type |
|-------------|---------------|-----------|
| `@Entity` class | `db_table` row | `maps_to_table` |
| `@Column` field | `db_column` row | `maps_to_column` |
| `@JoinColumn` | `db_foreign_key` row | `maps_to_fk` |

Both sides must reference the same `export_run_id` (or compatible snapshot timestamps).

## OpenRewrite recipe inputs (planned)

Recipes may read:

- Schema SSOT: table/column names, FK graph for relationship migrations
- Java AST SSOT: type hierarchy, EJB/XML crosswalk for recipe targeting

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
