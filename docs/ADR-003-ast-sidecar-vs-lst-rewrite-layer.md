# ADR-003: AST core + optional sidecars vs LST rewrite layer

**Status:** Accepted  
**Date:** 2026-06-27  
**Context:** Phase 2 stores code structure in `java-ast-ssot` using JavaParser AST. OpenRewrite's **Lossless Semantic Tree (LST)** is the natural choice for codemods. A common question is whether LST is "AST plus comments," and whether we can bolt LST capabilities onto the AST SSOT instead of maintaining two parse models.

This ADR complements [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md) (core vs profiles) and publicizes the AST/LST split first recorded in private lab-notes ADR-001 and [DUKESBANK-DEMO.md § AST vs LST](DUKESBANK-DEMO.md#design-decision-ast-vs-lst-vs-xml).

## Decision

**SSOT = semantic AST core + optional sidecar layers.**  
**LST = transform-time engine input only** — parsed on demand in `rewrite-recipes`, not stored as long-term SSOT.

Do **not** replicate a full OpenRewrite LST inside `java-ast-ssot`. Add **sidecars** only where analysis and verification need extra source fidelity.

### What LST adds beyond typical AST

LST is **not** "AST + a comment table." Relative to a normalized AST used for DRG queries:

| Dimension | AST (JavaParser, SSOT core) | LST (OpenRewrite, rewrite-time) |
|-----------|----------------------------|----------------------------------|
| Semantic structure | Yes — types, methods, calls, imports | Yes — oriented toward recipes |
| Comments | Often missing or weakly attached to nodes | In each node's **prefix/suffix**, bound to source bytes |
| Whitespace / formatting | Usually discarded | **Preserved** — lossless round-trip |
| Reversibility | Structure queryable; cannot guarantee byte-identical reprint | **Lossless** — unchanged regions keep original style |
| Type semantics | Partial (JavaParser resolution) | Rich **JavaType** attribution for rewrite safety |
| Primary goal | Analysis, DRG, crosswalk, verification | **Codemods** and recipe application |

Comments in LST travel with the **token stream** (prefix/suffix), not as stable semantic edges. A `//` may annotate the next line or a whole block — there is no universal comment→statement ownership model suitable for normalized SSOT.

### Layered model

```
┌──────────────────────────────────────────┐
│  java-ast-ssot CORE (always)             │
│  JavaParser → types, methods, DRG, IDs   │
└──────────────────┬───────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
 stack profile   sidecar(s)   rewrite LST
 (ADR-002)       (optional)   (on demand)
 javaee-ejb2-     source_      OpenRewrite in
 jboss, …         comment,     rewrite-recipes;
                  source_span  not in SQLite SSOT
```

| Layer | Repo / tool | Persisted in SSOT? | Role |
|-------|-------------|-------------------|------|
| **Core AST** | `java-ast-ssot` | Yes | DRG, stable IDs, profile crosswalk inputs |
| **Stack profile** | `java-ast-ssot` profiles | Yes (profile tables) | EJB/XML bindings, stack-specific edges |
| **Comment sidecar** | `java-ast-ssot` core export | Yes (`source_comment`) | Searchable comment blocks; no v1 semantic links |
| **Span sidecar** | `java-ast-ssot` (optional future) | Maybe | Node ↔ source range; still not full LST |
| **LST rewrite** | `rewrite-recipes` + OpenRewrite | **No** | Parse source at transform time; apply recipes |

### Sidecar rules (v1)

1. **`source_comment`** — store raw comment blocks `(file, start_line, end_line, kind, text)`. Do **not** attach comments to AST statement nodes on the parity critical path.
2. **Optional v2 heuristics** — Javadoc-before-declaration, previous-line `//`, etc., with explicit `confidence = heuristic`; never required for schema/code parity.
3. **No full LST clone in SQLite** — prefix/suffix on every node, import ordering trivia, and byte-exact reproduction belong in OpenRewrite's model, not in the DRG store.

### Two parse models, one source tree

The same legacy source directory may be parsed twice for different jobs:

| Job | Parser | When |
|-----|--------|------|
| Export DRG / crosswalk / analysis | JavaParser (+ profile XML) | `java-ast-ssot export` |
| Apply codemods | OpenRewrite LST | `rewrite-recipes` / Maven plugin at transform time |

**Single source of truth for bytes on disk:** the checked-out source tree (pinned by commit or export metadata). SSOT SQLite holds **derived, queryable structure**; LST is a **volatile, rewrite-optimized view** rebuilt when needed.

### Why not "AST + plugin = LST"?

A plugin on AST can recover **some** LST-like capabilities (comments, spans, optional file text hash) but cannot replace LST for rewrite without reimplementing OpenRewrite's lossless tree. That would duplicate maintenance, blur SSOT query semantics, and still fall short of recipe-grade type attribution.

**Accept the dual-parse trade-off:** simpler SSOT schema and deterministic DRG on one side; battle-tested lossless rewrite on the other.

## Consequences

### Positive

- Clear separation: **query SSOT** vs **transform engine**
- Sidecars extend fidelity without polluting core entity tables
- Aligns with [DEVELOPMENT-MODEL.md](DEVELOPMENT-MODEL.md) deterministic-core principle
- Same pattern applies to future `{language}-ast-ssot` repos — LST (or equivalent) stays in language-specific rewrite tooling

### Negative / trade-offs

- Two parse models in the program; recipes must not assume SSOT contains formatting or comment ownership
- Comment-aware migrations may need sidecar + heuristics or re-parse via LST at rewrite time
- `source_comment` / `source_span` sidecars add schema and export complexity when implemented

## Alternatives considered

| Option | Why not |
|--------|---------|
| LST as primary SSOT | Optimized for transform, not graph queries; comment/whitespace model unsuitable for normalized DRG |
| Full LST serialized into SQLite | Large, awkward to query; duplicates OpenRewrite; high maintenance |
| AST only, no sidecars, no LST | Cannot do lossless codemods or reliable comment search |
| Spoon or other unified model | Heavier; JavaParser + OpenRewrite split matches our extract vs transform layers |

## Implementation plan

| Step | Deliverable | Status |
|------|-------------|--------|
| 1 | Document layered model (this ADR + DUKESBANK-DEMO cross-links) | Done |
| 2 | `source_comment` table + export in `java-ast-ssot` | Done |
| 3 | `rewrite-recipes` repo — LST parse at recipe apply time | ✅ Alpha (3.0–3.3 + ADR-008 L1/L2/L3) |
| 4 | Optional `source_span` sidecar | 💡 Idea |

## References

- [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md) — core vs stack profiles
- [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md) — DRG design; § AST vs LST vs XML
- [ARCHITECTURE.md](ARCHITECTURE.md) — extraction vs transformation layers
- [SSOT-SCHEMA.md](SSOT-SCHEMA.md) — core vs extension contracts
- [ADR-004](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md) — crosswalk mapping roles and edge kinds
- Private lab-notes ADR-001 — original AST-not-LST-for-SSOT decision
