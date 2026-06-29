# ADR-009: Rewrite engine port, presets, and run manifest

**Status:** Accepted  
**Date:** 2026-06-28  
**Context:** Phase 3 delivered individual Anchor recipes (ADR-007 stack migration, ADR-008 L1) plus upstream OpenRewrite building blocks. Consumers need a **swappable rewrite engine** and **config-driven recipe chains** — not hardcoded `activeRecipes` in every `pom.xml`.

Complements [ADR-003](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md) (LST at apply time), [ADR-007](ADR-007-rewrite-recipes-session-and-cmp-jpa.md), [ADR-008](ADR-008-java-language-modernization-and-tuple-lists.md).

---

## Decision

### 1. Three recipe layers

| Layer | Source | Activation |
|-------|--------|------------|
| **Upstream** | OpenRewrite BOM, community recipes | Reference by FQN in YAML (`org.openrewrite.java.ChangeType`, etc.) |
| **Anchor** | `rewrite-recipes` Java + YAML composites | Registered via SPI and `META-INF/rewrite/*.yml` |
| **Presets** | `META-INF/rewrite/presets/*.yml` | Ordered `recipeList`; selected by name at run time |

Presets **compose** upstream + Anchor recipes. Order is significant and documented per preset.

### 2. Default activation via Maven property

`rewrite-recipes/pom.xml` uses:

```xml
<anchor.rewrite.preset>com.anchor.migration.presets.Smoke</anchor.rewrite.preset>
...
<recipe>${anchor.rewrite.preset}</recipe>
```

Override with `-Danchor.rewrite.preset=...` or consumer-project property. **Do not** hardcode migration stacks in plugin config.

### 3. YAML-first for mechanical L1 chains

ADR-008 L1 (`Vector`, `Hashtable`, `StringBuffer`) is declared as YAML composite `LanguageModernizationL1`, delegating to upstream `ChangeType`. Java wrapper `VectorToArrayList` remains for backward-compatible SPI and narrow tests; new presets reference the YAML composite.

### 4. RewriteEngine remains swappable

OpenRewrite is the **reference implementation** of the rewrite port ([ADR-003](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md)). Presets and recipe FQNs are engine-facing configuration — a future non-OpenRewrite adapter would map preset names to its own recipe catalog without changing SSOT or crosswalk contracts.

**Run manifest (future):** A JSON/YAML manifest beside linked SSOT will record `{ preset, recipeVersions, targetModules, requiresLinkedDb }` for reproducible runs. v0.1 presets embed `recipeFamily` and `requiresLinkedDb` in YAML as forward-compatible metadata.

---

## Preset catalog (v0.1)

| Preset | Chain |
|--------|-------|
| `com.anchor.migration.presets.Smoke` | `AddAnchorProbeComment` |
| `com.anchor.migration.presets.LanguageL1Only` | `LanguageModernizationL1` |
| `com.anchor.migration.presets.LanguageL2Only` | `LanguageModernizationL2` — homogeneous raw `ArrayList` typing |
| `com.anchor.migration.presets.DukesBankStackMigration` | L1 → `SessionBeanToSpringService` → `CmpScalarEntityToJpa` |

Duke's Bank stack order matches ADR-007 + ADR-008 recommended run order.

---

## Consequences

**Positive**

- Consumers switch migration depth with one property.
- Upstream recipe upgrades flow through YAML without Java recompilation where mechanical.
- Preset catalog is testable (`PresetCatalogTest` resolves all names from classpath).

**Negative / follow-ups**

- Run manifest + SSOT-linked targeting not implemented yet (M2+).
- `rewrite:run` against external repos still requires manual `pom` wiring; CLI wrapper planned.

---

## Verification

| Artifact | Location |
|----------|----------|
| Preset YAML | `rewrite-recipes/src/main/resources/META-INF/rewrite/presets/` |
| L1 composite | `META-INF/rewrite/language-modernization-l1.yml` |
| Docs | [rewrite-presets.md](https://github.com/anchor-migration/rewrite-recipes/blob/main/docs/rewrite-presets.md) |
| Tests | `PresetCatalogTest`, `LanguageModernizationL1Test` |
