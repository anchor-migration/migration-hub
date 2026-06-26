# ADR-002: Java AST SSOT — core vs stack profiles

**Status:** Accepted  
**Date:** 2026-06-27  
**Context:** `java-ast-ssot` v0.1 POC was built against Duke's Bank (EJB 2.x CMP + JBoss XML). EJB-specific tables and parsers were added to the same schema and CLI as generic Java AST extraction, which blurred product boundaries.

## Decision

**`java-ast-ssot` is a generic Java source AST exporter.** Duke's Bank is a **reference application** for end-to-end validation, not the product scope.

Legacy **stack** concerns (Java EE EJB 2.x, Spring, JPA, etc.) are **optional profiles / extensions**, analogous to dialect adapters in `db-metadata`.

### Layering

| Layer | Scope | Always on? |
|-------|--------|------------|
| **Core** | Java sources → types, methods, fields, imports, source files | Yes |
| **Stack profile** | Deployment descriptors and stack-specific bindings | No — enabled by profile or auto-detect |
| **Crosswalk** | Edges linking code SSOT ↔ schema SSOT or across stack entities | Profile-dependent |

### v0.1 POC (current code, pre-refactor)

Implementation still monolithic. Treat as:

- **Core:** JavaParser path (any `--source-root` with `.java` files)
- **Implicit profile:** `javaee-ejb2-jboss` — always parses `ejb-jar.xml` and `jbosscmp-jdbc.xml` when present

This is **technical debt**; Step 2 refactor will split core and profiles explicitly.

### Target CLI (after refactor)

```bash
# Core only — any Java project
java-ast-ssot export --source-root /path/to/src --out metadata/java.db

# Enable Java EE EJB 2.x + JBoss CMP profile (Duke's Bank)
java-ast-ssot export --source-root /path/to/bank \
  --profile javaee-ejb2-jboss \
  --out metadata/dukesbank-code.db
```

Profiles may also be **auto-detected** from known descriptor files under `--source-root`, but explicit `--profile` must remain for reproducible CI.

### Duke's Bank role

| Question | Answer |
|----------|--------|
| Is `java-ast-ssot` for Duke's Bank only? | **No** |
| What is Duke's Bank? | Canonical **demo** for schema + Java DRG + crosswalk |
| Why so much EJB in v0.1? | First stack profile, not the permanent core API |

### Heterogeneous migration (COBOL → Java)

- `cobol-ast-ssot` (future) = source language SSOT
- `java-ast-ssot` = **target language SSOT** (modern Spring or legacy EJB Java both use the Java core)
- Cross-language linking = separate layer, not embedded in either language repo

## Schema direction (refactor)

**Core tables (stable):** `export_run`, `source_file`, `java_type`, `java_method`, `java_field`, `java_import`

**Profile tables (v0.1 today, to be namespaced):** `ejb_bean`, `ejb_cmp_field`, `ejb_ref`, profile-specific `crosswalk_edge` kinds

Future profiles add tables or use `extension_*` / `artifact_type` columns — see [SSOT-SCHEMA.md](SSOT-SCHEMA.md).

## Consequences

**Positive**

- Same repo serves Spring modernization and Java EE legacy paths
- Clear analogy to `db-metadata` dialects
- Room for `cobol-ast-ssot` without overloading `java-ast-ssot`

**Negative / cost**

- Refactor required before adding Spring/JPA profiles cleanly
- Docs must distinguish core vs profile until refactor ships

## Implementation plan

| Step | Deliverable | Status |
|------|-------------|--------|
| 1 | Document positioning (this ADR + hub docs + `java-ast-ssot` README) | In progress |
| 2 | Code refactor: `Profile` SPI, `--profile`, split schema, tests on core-only export | Planned |

## References

- [ARCHITECTURE.md](ARCHITECTURE.md) — extraction layer
- [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md) — reference app runbook
- [SSOT-SCHEMA.md](SSOT-SCHEMA.md) — core vs extension contracts
