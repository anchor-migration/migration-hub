# ADR-010: anchor-stubborn as horizontal LLM context

**Status:** Accepted  
**Date:** 2026-06-29  
**Last updated:** 2026-07-01 (beta `0.9.0b1`)  
**Context:** [anchor-stubborn](https://github.com/anchor-migration/anchor-stubborn) is an independent **code context compiler** (SCIP → symbol graph → privacy-safe stub text). The migration program already has full-depth SSOT tools (`java-ast-ssot`, `db-metadata`) and human review (`anchor-explorer`). Teams also need **token-bounded LLM context** when drafting mappings, recipe designs, or PR reviews — without shipping raw source bodies to models.

Complements [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md) (full AST SSOT), [ADR-003](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md) (LST at rewrite time), [ADR-005](ADR-005-multi-tier-alignment-and-ssot-explorer.md) (human Explorer UI).

---

## Decision

### 1. Horizontal capability, not a pipeline layer

`anchor-stubborn` is **not** Layer 1–4 in [ARCHITECTURE.md](ARCHITECTURE.md). It sits **above** the migration pipeline as shared infrastructure:

```
                    ┌─────────────────────┐
                    │   anchor-stubborn   │
                    └──────────┬──────────┘
                               │ stub / anchor-dsl text
     ┌─────────────────────────┼─────────────────────────┐
     ▼                         ▼                         ▼
 LLM mapping draft      rewrite-recipes design      PR diff CI
```

The repo has its **own release cycle, CI, and consumers** beyond migration. `migration-hub` documents the integration contract; it does not own Stubborn's product scope.

### 2. When migration uses Stubborn

Before asking an LLM to propose rewrite mappings or recipe changes:

```bash
# 1. Generate SCIP (e.g. scip-java in the target repo)
# 2. Index
anchor-stubborn index --scip index.scip --out metadata/code-context.db

# 3. Context for a migration target class (Java stub — familiar to codegen models)
anchor-stubborn context metadata/code-context.db \
  --target "<stable_id>" \
  --max-tokens 12000 \
  --out /tmp/target.stub.java

# Or compact graph format (fewer tokens; v0.7+)
anchor-stubborn context metadata/code-context.db \
  --target "<stable_id>" \
  --format anchor-dsl \
  --out /tmp/target.anchor-dsl

# 4. Optional KPI for runbooks / blog
anchor-stubborn metrics metadata/code-context.db \
  --target "<stable_id>" \
  --sources /path/to/src/main/java
```

Feed the output to the LLM instead of raw sources. For `anchor-dsl`, use the [LLM prompt snippet](https://github.com/anchor-migration/anchor-stubborn/blob/main/docs/ANCHOR-DSL-LLM.txt) or the embedded `# Guide` header in each block.

**Agents:** run `anchor-stubborn mcp` (stdio) — tools `get_context`, `list_symbols`, `metrics`. See [MCP.md](https://github.com/anchor-migration/anchor-stubborn/blob/main/docs/MCP.md).

Reference examples:

- [migration-bridge](https://github.com/anchor-migration/anchor-stubborn/tree/main/examples/migration-bridge) — Duke's Bank–style workflow
- [demo-spring](https://github.com/anchor-migration/anchor-stubborn/tree/main/examples/demo-spring) — primary in-repo E2E
- [spring-petclinic](https://github.com/anchor-migration/anchor-stubborn/tree/main/examples/spring-petclinic) — scale-up validation

**Baseline KPIs (Docker E2E, pinned toolchains):**

| Example | Target | Token savings | Notes |
|---------|--------|---------------|-------|
| demo-spring | `OrderService` | ~81% | [order-service-context.md](https://github.com/anchor-migration/anchor-stubborn/blob/main/examples/demo-spring/cases/order-service-context.md) |
| demo-spring | `OrderController` | ≥75% | Web → service case |
| demo-spring | `OrderService#payOrder` | ~80% | Method-level payment flow |
| spring-petclinic | `VetController` | ~90% | ~375 index symbols; weekly CI |

### 3. When migration does NOT use Stubborn

| Need | Tool |
|------|------|
| Full AST export for Explorer / crosswalk | `java-ast-ssot` |
| Schema ↔ code linking | `java-ast-ssot crosswalk` + `db-metadata` |
| Applying rewrites | `rewrite-recipes` + OpenRewrite |
| Structural parity | `parity-verify` |
| Schema reconciliation | `db-metadata verify` |

Stubborn **does not replace** `java-ast-ssot`. It answers a different question: *"What does the AI need right now?"* vs *"What's in this project?"*

### 4. Weak coupling rules

1. **No compile-time dependency** from `anchor-stubborn` → `rewrite-recipes`, `java-ast-ssot`, or `db-metadata`
2. **Shared conventions only**: stable IDs, SQLite snapshot pattern, reconcile vocabulary ([SSOT-SCHEMA.md](SSOT-SCHEMA.md) symbol-graph section)
3. **Optional** in any migration runbook — teams may use `java-ast-ssot` alone when full AST is required
4. **Privacy contract**: output includes declarations and signatures; excludes method bodies and business literals in annotation values

### 5. Symbol-graph SSOT (Stubborn-owned)

Stubborn maintains a separate SQLite schema for SCIP symbol graphs — not merged into Java AST SSOT. DDL: [anchor-stubborn/.../v1.sql](https://github.com/anchor-migration/anchor-stubborn/blob/main/src/anchor_stubborn/store/schema/v1.sql). Documented in [SSOT-SCHEMA.md](SSOT-SCHEMA.md).

### 6. CI integration (shipped)

- `anchor-stubborn diff` — symbol reconcile between two indexes (exit 1 on missing symbols)
- [pr-symbol-diff.yml](https://github.com/anchor-migration/anchor-stubborn/blob/main/.github/workflows/pr-symbol-diff.yml) — PR guard workflow
- demo-spring Docker E2E on every PR; spring-petclinic scale-up weekly / manual

---

## Consequences

- Program docs (`README`, `START-HERE`, `ARCHITECTURE`, `ROADMAP`) list `anchor-stubborn` as a sibling repo with **horizontal** role
- `anchor-migration.code-workspace` includes the repo for local multi-root work
- MCP server and Anchor-DSL weaver are **Stubborn deliverables** (v0.4 / v0.7); migration-hub references them in runbooks but does not implement them
- Java-first **beta** at **`0.9.0b1`** — [BETA.md](https://github.com/anchor-migration/anchor-stubborn/blob/main/docs/BETA.md)

## References

- [anchor-stubborn README](https://github.com/anchor-migration/anchor-stubborn)
- [POSITIONING.md](https://github.com/anchor-migration/anchor-stubborn/blob/main/docs/POSITIONING.md)
- [INTEGRATION.md](https://github.com/anchor-migration/anchor-stubborn/blob/main/docs/INTEGRATION.md) (repo-local detail; this ADR is the program SSOT)
- [ANCHOR-DSL.md](https://github.com/anchor-migration/anchor-stubborn/blob/main/docs/ANCHOR-DSL.md)
