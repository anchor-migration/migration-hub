# ADR-004: Crosswalk contract — mapping roles and edge kinds

**Status:** Accepted  
**Date:** 2026-06-27  
**Context:** Phase 2 produces two SSOT snapshots: schema (`db-metadata`) and code (`java-ast-ssot`). Linking them requires **crosswalk** edges from Java (or stack-specific) entities to database tables and columns.

Binding signals differ by stack: Java EE CMP uses deployment XML; JPA uses annotations; MyBatis uses mapper XML or `@Select` SQL. Projects also differ in **mapping shape**: some types map 1:1 to a physical table; others are **read models** backed by JOIN SQL with no single owning table.

Profile-specific extractors must not each invent incompatible edge semantics. This ADR defines a **unified crosswalk contract** that profiles populate and a **link step** validates against schema SSOT.

Complements [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md) (profiles as binding extractors) and [SSOT-SCHEMA.md](SSOT-SCHEMA.md) (stable IDs on both sides).

## Decision

### Two layers

| Layer | Owner | Output | When |
|-------|-------|--------|------|
| **Profile extract** | `java-ast-ssot` profiles | Stack-specific **binding facts** in namespaced profile tables | `export --profile …` |
| **Crosswalk link** | `java-ast-ssot crosswalk` (planned) | Normalized **`code_schema_link`** rows pinned to both `export_run_id`s | After code + schema export |

Schema SSOT describes **physical database** objects only. Code SSOT core describes **Java structure** only. Profiles describe **how the stack connects code to persistence**. The link step merges profile facts + AST IDs + schema rows into one queryable graph.

```
Profile extract (stack-specific)     Crosswalk link (stack-neutral)
────────────────────────────────     ────────────────────────────────
javaee-ejb2-jboss: XML facts    ──►  code_schema_link
jpa: annotation facts           ──►  + mapping_role
mybatis: mapper/SQL facts       ──►  + validation vs db_table/db_column
```

Existing `javaee_ejb2_jboss_crosswalk` (`ejb_to_table`, `java_type_to_ejb`) is an **intermediate** profile artifact. The link step **normalizes** these into canonical edge kinds (see below).

### Mapping roles

Every Java type (or mapper result type) that participates in persistence crosswalk carries a **`mapping_role`**. Roles drive parity and recipe expectations.

| Role | Meaning | Typical examples | Schema link pattern |
|------|---------|------------------|---------------------|
| **`persistent_entity`** | Mutable/read-write object backed by a primary table | EJB CMP entity, JPA `@Entity`, MyBatis `resultMap` → single table | `type_maps_to_table` + `field_maps_to_column` |
| **`read_model`** | Read-only projection; no single owning table | MyBatis JOIN → DTO, `@SqlResultSetMapping`, report VO | `type_backed_by_sql` + `field_maps_to_column_via` + `sql_references_table` |
| **`transfer_object`** | DTO / wire type with no direct ORM binding | `AccountDetails`, WS payload | Usually **no** table edges; optional field references |
| **`repository_boundary`** | DB access surface | DAO interface, MyBatis mapper method | Edges on **method/SQL**, not always on the DTO |

**Parity policy:**

- **`persistent_entity`:** structural parity on critical path (type ↔ table, field ↔ column).
- **`read_model`:** SQL/behavior parity on critical path; structural edges are supporting metadata.
- **`transfer_object`:** not required to map to a table.

### Canonical edge kinds (`edge_kind`)

Profiles emit facts; the link step writes these **canonical** kinds into `code_schema_link`:

| edge_kind | source_stable_id | target_stable_id | Use |
|-----------|------------------|------------------|-----|
| `type_maps_to_table` | `{package}.{Type}` | `{schema}.{table}` | Persistent entity primary table |
| `field_maps_to_column` | `{Type}#{field}` | `{schema}.{table}.{column}` | Single-table column binding |
| `field_maps_to_column_via` | `{Type}#{field}` | `{schema}.{table}.{column}` | Column from JOIN/alias/subquery (read model) |
| `type_backed_by_sql` | `{package}.{Type}` | `sql:{mapper}#{statementId}` | Read model with no single table |
| `method_executes_sql` | `{Type}#{method}(…)` | `sql:{mapper}#{statementId}` | Mapper/DAO method → SQL artifact |
| `sql_references_table` | `sql:{mapper}#{statementId}` | `{schema}.{table}` | Tables touched by a SQL statement |
| `relationship_maps_to_table` | `ejb:{name}` or `{Type}` | `{schema}.{table}` | CMR / `@ManyToMany` / xref table |
| `stack_bridge` | Profile-specific | Profile-specific | Intermediate IDs (e.g. `java_type → ejb:{name}`); optional in linked output |

Profile-local kinds (e.g. `ejb_to_table`, `java_type_to_ejb`) are **normalized** during link:

```
java_type_to_ejb  +  ejb_to_table  →  type_maps_to_table (mapping_role = persistent_entity)
cmp_field row     +  schema column →  field_maps_to_column
```

### Link row metadata

Each `code_schema_link` row includes:

| Column | Purpose |
|--------|---------|
| `edge_kind` | Canonical kind (table above) |
| `source_stable_id` | Code-side ID |
| `target_stable_id` | Code-side, schema-side, or SQL artifact ID |
| `mapping_role` | `persistent_entity` \| `read_model` \| … |
| `profile_id` | e.g. `javaee-ejb2-jboss`, `jpa`, `mybatis` |
| `binding_source` | `jbosscmp_xml`, `ejb_jar_xml`, `jpa_annotation`, `mybatis_xml`, `mybatis_annotation`, … |
| `evidence_ref` | File path, mapper id, XPath, annotation line — for audit |
| `confidence` | `authoritative` \| `inferred` |
| `code_export_run_id` | Pin code snapshot |
| `schema_export_run_id` | Pin schema snapshot |

**Confidence:** XML deployment descriptors and explicit JPA/MyBatis mappings are `authoritative`. Regex/SQL-parse table extraction from dynamic SQL is `inferred` — usable for analysis, not parity-critical without review.

### Stable ID extensions

Existing IDs from [SSOT-SCHEMA.md](SSOT-SCHEMA.md) plus:

| Entity | Format | Example |
|--------|--------|---------|
| SQL statement | `sql:{mapperRelativePath}#{statementId}` | `sql:mapper/AccountMapper.xml#listAccountSummary` |
| SQL (annotation) | `sql:{Type}#{methodName}` | `sql:com.example.AccountMapper#findAll` |
| EJB (intermediate) | `ejb:{ejbName}` | `ejb:AccountBean` |

Schema targets continue to use `db-metadata` keys: `{schema}.{table}`, `{schema}.{table}.{column}`.

### Stack profiles — binding signals

#### `javaee-ejb2-jboss` (implemented, link planned)

| Signal | Role | Canonical edges |
|--------|------|-----------------|
| `ejb-jar.xml` entity/session | bridge | `stack_bridge`: `java_type → ejb:{name}` |
| `jbosscmp-jdbc.xml` entity/table | `persistent_entity` | `type_maps_to_table`, `field_maps_to_column` |
| EJB CMR / xref | `relationship_maps_to_table` | e.g. `CUSTOMER_ACCOUNT_XREF` (planned) |

Duke's Bank: MySQL seed has **no FK constraints**; relationship edges come from EJB/XML, not schema SSOT alone.

#### `jpa` (planned)

| Signal | Role | Canonical edges |
|--------|------|-----------------|
| `@Entity` + `@Table` | `persistent_entity` | `type_maps_to_table` |
| `@Column` / field name default | | `field_maps_to_column` |
| `@JoinColumn`, `@ManyToMany` | | `relationship_maps_to_table` or FK target |
| `@SqlResultSetMapping`, native query DTO | `read_model` | `type_backed_by_sql`, `field_maps_to_column_via` |

#### `mybatis` (planned)

| Signal | Role | Canonical edges |
|--------|------|-----------------|
| `resultMap` + single-table CRUD | `persistent_entity` | `type_maps_to_table`, `field_maps_to_column` |
| `@Results` / `@Result` annotations | `persistent_entity` | same |
| `<select>` with JOIN → DTO | `read_model` | `type_backed_by_sql`, `sql_references_table`, `field_maps_to_column_via` |
| Mapper interface method | `repository_boundary` | `method_executes_sql` |

**Do not** collapse a JOIN-backed DTO into `type_maps_to_table` for one arbitrary table — that loses JOIN semantics and breaks safe refactors.

### Crosswalk link CLI (planned)

```bash
java-ast-ssot crosswalk \
  --code-db metadata/dukesbank-code.db \
  --schema-db metadata/dukesbank.db \
  --db-schema dukesbank \
  --out metadata/dukesbank-linked.db
```

Link step responsibilities:

1. Read profile facts + core AST IDs from code SSOT.
2. Read `db_table` / `db_column` from schema SSOT.
3. Normalize profile edges → canonical `edge_kind`.
4. **Validate** targets exist in schema SSOT (table/column name case per dialect).
5. Emit `code_schema_link` with both export run IDs.
6. Report unresolved references (missing table, column mismatch).

`--db-schema` replaces hardcoded schema name in profile IDs (see `JavaEeEjb2JbossIds.DEFAULT_DB_SCHEMA`).

### Output location

**v1:** `code_schema_link` lives in the **linked output SQLite** produced by `crosswalk` (third file), referencing rows in the two input DBs by stable ID + export_run_id. Avoid duplicating full schema/code DDL in v1.

**v2 (optional):** embed links in code SSOT if tooling prefers single artifact — requires schema version negotiation.

## Consequences

### Positive

- One query model for recipes, parity, and AI agents across EJB, JPA, and MyBatis
- Read models and 1:1 entities handled explicitly — no false table parity
- Profiles stay thin: extract facts; link step owns normalization and validation
- Duke's Bank EJB path maps cleanly onto `persistent_entity` without redesign

### Negative / cost

- Link step and `code_schema_link` DDL must be implemented and tested
- MyBatis SQL parsing adds complexity; dynamic SQL may stay `inferred` only
- Multiple profiles in one project require merge rules (same type, conflicting roles → error)

## Alternatives considered

| Option | Why not |
|--------|---------|
| Each profile owns final schema links in its tables | Incompatible edge kinds; duplicate validation; hard for cross-profile queries |
| Assume every `@Entity`-like type maps 1:1 to a table | Fails MyBatis JOIN DTOs and many legacy report objects |
| Put crosswalk only in schema SSOT repo | Code-side binding evidence lives in Java/XML; wrong ownership |
| Single SQLite for Java+schema | Complicates independent re-export and multi-app schema sharing |

## Implementation plan

| Step | Deliverable | Status |
|------|-------------|--------|
| 1 | Document contract (this ADR + SSOT-SCHEMA update) | Done |
| 2 | `crosswalk` CLI + `code_schema_link` DDL; Duke's Bank EJB `persistent_entity` | Done (1.0.0-SNAPSHOT) |
| 3 | Normalize `javaee_ejb2_jboss_*` → canonical edges; `--db-schema` on link | Done |
| 4 | `jpa` profile + link | 📋 Planned |
| 5 | `mybatis` profile: SQL artifacts + `read_model` edges | 📋 Planned |
| 6 | EJB relationship → xref table edges (Duke's Bank) | 💡 Idea |

## References

- [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md) — profiles as stack adapters
- [SSOT-SCHEMA.md](SSOT-SCHEMA.md) — stable IDs; linking section
- [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md) — crosswalk example; dual SSOT diagram
- [ARCHITECTURE.md](ARCHITECTURE.md) — extract → link → transform
