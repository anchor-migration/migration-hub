# ADR-005: Multi-tier mapping alignment, edge coloring, and SSOT Explorer

**Status:** Accepted  
**Date:** 2026-06-27  
**Context:** Anchor Migration SSOT captures schema, code structure, and crosswalk links (ADR-004). Stakeholders cannot infer value from SQLite alone. Real legacy systems also map across **multiple tiers** — not only DB ↔ persistence entity, but data entity ↔ domain entity ↔ UI — with **name drift** and **type mismatches** that a binary “linked / not linked” model hides.

This ADR defines:

1. **Multi-tier mapping** as a first-class concept extending crosswalk.  
2. **Alignment quality** and **edge coloring** for human review and visualization.  
3. **`anchor-explorer`** as a **first-class human interface** — not a disposable demo sketch.

Complements [ADR-004](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md) (persistence crosswalk) and [ARCHITECTURE.md](ARCHITECTURE.md) (extract → link → transform → verify).

## Decision

### 1. Mapping is a chain, not a single hop

Ideal projects achieve **1:1 alignment** across tiers:

```
DB column  ↔  data entity field  ↔  domain attribute  ↔  UI binding
(physical)     (persistence)         (domain)             (presentation)
```

| Tier | Typical legacy carriers | SSOT today |
|------|-------------------------|------------|
| **physical** | RDBMS tables/columns | `db-metadata` |
| **persistence** | EJB CMP, JPA `@Column`, MyBatis `resultMap` | `java-ast-ssot` + crosswalk |
| **domain** | Domain objects, service DTOs (when separate from persistence) | 📋 future |
| **presentation** | JSP forms, JSF, REST/JSON field names | 📋 future |

**Persistence crosswalk** (ADR-004) anchors **physical ↔ persistence**. Higher tiers use the same **`code_schema_link` edge model** extended with `mapping_tier` — not a separate ad-hoc graph.

Planned edge kinds (v2+):

| edge_kind | Tiers connected |
|-----------|-----------------|
| `field_maps_to_column` | persistence ↔ physical (implemented) |
| `field_maps_to_field` | domain ↔ persistence |
| `field_maps_to_ui_binding` | presentation ↔ domain |

### 2. Alignment quality — beyond linked / not linked

Even when two names are “linked,” quality varies. SSOT must record **why** an edge exists and **how trustworthy** it is.

#### Name drift classes

After **normalization** (case, separators: `account_id`, `ACCOUNT_ID`, `accountId` → canonical `accountid`), remaining differences are classified:

| Class | Meaning | Example | Typical per-direction color |
|-------|---------|---------|----------------------------|
| **`none`** | Normalized names match; no semantic drift | `account_id` ↔ `accountId` | **Green** (each direction, unless type says otherwise) |
| **`explainable`** | Understandable alias or abbreviation | `account` ↔ `acct` | **Yellow** |
| **`questionable`** | Plausible but needs human review | `balance` ↔ `annual_balance` | **Orange** |
| **`unexplainable`** | No reasonable semantic bridge | `userId` ↔ `orderId` | **Red** (both directions) |

**Convention-only changes** (case, snake/camel) collapse to **`none`** after normalization — they are not drift.

#### Type alignment — directional, not symmetric

Type mismatch is **directional**. Legacy systems often tolerate **lossy forward** paths but fail on **reverse** paths.

Typical read/display chain (usually tolerable):

```
DB INT  ──widening──►  Java Integer  ──widening──►  Domain long  ──stringified──►  UI String
```

Typical write/persist chain (where bugs hide):

```
UI String  ──parse──►  Domain long  ──narrowing──►  Integer  ──►  DB INT   ⚠ dangerous
```

| Direction | Examples | Default severity |
|-----------|----------|------------------|
| **Forward (read / display)** | `int` → `long`, `int` → `String` for formatting | Often **OK** — green or yellow |
| **Backward (write / persist)** | `String` → `long`, `long` → `int`, `String` → `int` | **Dangerous** — orange or red |

**“The return path is where it gets dangerous”** — Anchor Migration treats **backward / round-trip** type steps as first-class, not an afterthought.

Each edge records **two directed type relations** (when types are known):

| Field | Meaning |
|-------|---------|
| `type_relation_forward` | Lower tier → higher tier (read/display) |
| `type_relation_backward` | Higher tier → lower tier (write/persist) |

Shared vocabulary per direction:

| Value | Meaning |
|-------|---------|
| `exact` | Same logical type |
| `widening` | e.g. `int` → `long` |
| `narrowing` | e.g. `long` → `int` |
| `stringified` | numeric/date → `String` |
| `parsed` | `String` → numeric (explicit parse; lossy) |
| `incompatible` | no safe conversion |

**Round-trip summary** (optional derived label for filters/reports):

| `round_trip_class` | Condition |
|--------------------|-----------|
| `safe` | Both directions **green** |
| `asymmetric` | Forward green, backward yellow or worse (common for precision drift) |
| `unsafe_backward` | Backward **orange** or **red** on persistence ↔ physical |
| `incompatible` | Either direction **red** |

#### Bidirectional edge coloring (traffic-map model)

Crosswalk edges are **bidirectional** (`A ↔ B`). Alignment quality is **not** one color per edge — it is **one color per direction**, like a road map where one lane is clear and the other congested.

```
   physical (INT)  ══════►  persistence (long)     forward:  🟢 green  (precision widens — OK)
                   ◄══════                              backward: 🟡 yellow (narrowing — caution)

   exact match     ══════►                              forward:  🟢 green
                   ◄══════                              backward: 🟢 green
```

| Stored field | Meaning |
|--------------|---------|
| **`color_forward`** | Lower tier → higher tier (read / display / widen precision) |
| **`color_backward`** | Higher tier → lower tier (write / persist / narrow precision) |

Each direction is computed independently from **name drift** + **type relation on that direction**:

| Direction | Inputs | Examples |
|-----------|--------|----------|
| Forward | `name_drift_class` + `type_relation_forward` | `INT`→`long` widening + name `none` → **green** |
| Backward | `name_drift_class` + `type_relation_backward` | `long`→`INT` narrowing + name `none` → **yellow** (not red unless lossy/unsafe) |

**Rules:**

- **Full match** (name `none`, type `exact` both ways) → **`color_forward` = green, `color_backward` = green**.  
- **Precision upgrade one way only** (e.g. `int`↔`long`) → forward **green**, backward **yellow** — *widening forward is fine; narrowing on the return path needs caution*.  
- **`unexplainable` name** → **both directions red**.  
- **Parity-critical persist path** uses **`color_backward`** on persistence ↔ physical; forward-only green must not hide backward yellow/orange.

Explorer **must render both directions** on the same edge (split stroke, dual arrowheads, or hover “A→B / B→A” legend). Toggling “highlight write path” emphasizes `color_backward` without losing forward context.

#### Planned link metadata (extends ADR-004)

```text
name_drift_class:        none | explainable | questionable | unexplainable
type_relation_forward:   exact | widening | narrowing | stringified | parsed | incompatible
type_relation_backward:  exact | widening | narrowing | stringified | parsed | incompatible
color_forward:           green | yellow | orange | red
color_backward:          green | yellow | orange | red
round_trip_class:        safe | asymmetric | unsafe_backward | incompatible  -- optional summary
normalized_source:       accountid
normalized_target:       accountid
mapping_tier:            physical | persistence | domain | presentation
```

v1 Duke's Bank (CMP XML authoritative): persistence ↔ physical edges default to **`none` + green/green** when schema column and types align.

### 3. Edge coloring in visualization

**Edge coloring** is **bidirectional** — the primary visual language in Explorer.

| Color | Drift / type (per direction) | User action |
|-------|------------------------------|-------------|
| 🟢 **Green** | Name `none` + safe type on **this** direction | Trust for that direction |
| 🟡 **Yellow** | Explainable name and/or narrowing/parsed on **this** direction | Caution; document or review |
| 🟠 **Orange** | Questionable name or unsafe backward persist | Review queue |
| 🔴 **Red** | Unexplainable name or incompatible type on **this** direction | Block automation |

**Same edge, two colors** — e.g. forward green + backward yellow for `int`↔`long`. Filters: “any backward orange+”, “both directions green”.

Legend, bidirectional rendering, and drill-down to **`evidence_ref`** / **`crosswalk_issue`** are **required** — not optional polish.

### 4. SSOT Explorer — first-class human interface

Explorer exists to **communicate** SSOT value, but as long as humans use Anchor Migration it is a **product interface**, not a throwaway script.

#### Repository

New repo: **`anchor-explorer`** (public under anchor-migration org).

```
db-metadata export ──┐
java-ast-ssot export ├──► SQLite snapshots ──► anchor-explorer (read-only)
crosswalk link ──────┘
```

#### Architectural rules

| Rule | Rationale |
|------|-----------|
| **Read-only** | Never mutates SSOT; not a source of truth |
| **Separate repo** | UI iteration must not coupling to Java/Python CLIs |
| **Pinned snapshots** | UI displays `export_run_id`, tool versions, paths |
| **Layered app** | SSOT readers → view model → components (testable) |
| **Accessibility** | WCAG-oriented contrast for green/yellow/orange/red; not color-only |
| **No “demo quality” UX** | Clear nav, loading/error states, deep links to entities |
| **Deterministic data layer** | Same `.db` inputs → same graph metrics (UI chrome may evolve) |

#### MVP views (Duke's Bank hero)

1. **Schema** — ER diagram of 5 tables  
2. **Code** — packages / EJB entities  
3. **Crosswalk** — persistence ↔ physical with **bidirectional edge colors** (traffic-map edges)  
4. **Issues** — `crosswalk_issue` + drift class summary  
5. **Comments** (optional) — `source_comment` browse by file  

v0 may ship static JSON generated from SQLite; v1+ uses in-browser SQLite or a thin local server — **without** lowering UX or architecture standards.

#### Non-goals for Explorer

- Replacing CLIs or editing SSOT  
- Running migration recipes  
- Being the only documentation (hub ADRs remain canonical)

## Consequences

### Positive

- Stakeholders see **quality of mapping**, not just existence of edges  
- Architects can extend to domain/UI tiers without a new paradigm  
- Color + drift class unify review queues, parity gates, and blog demos  
- Explorer treated seriously improves adoption and contributor onboarding

### Negative / cost

- Alignment scoring adds rules, tests, and manual review workflows  
- Domain/UI tier extraction is stack-specific (future profiles)  
- Explorer is ongoing UX/engineering investment — not a one-off HTML file

## Implementation plan

| Step | Deliverable | Status |
|------|-------------|--------|
| 1 | Document tiers, drift classes, colors (this ADR) | Done |
| 2 | Add alignment metadata + **`color_forward` / `color_backward`** to link rows (Java) | Done (1.0.0-SNAPSHOT) |
| 3 | **`anchor-explorer` repo** — MVP crosswalk graph + legend | 🚧 Alpha |
| 4 | Domain / presentation tier profiles + edges | 💡 Idea |
| 5 | Parity gate uses red / incompatible rules | 💡 Idea |

## References

- [ADR-004](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md) — crosswalk contract  
- [SSOT-SCHEMA.md](SSOT-SCHEMA.md) — stable IDs; alignment metadata (planned)  
- [DUKESBANK-DEMO.md](DUKESBANK-DEMO.md) — reference demo  
- [ARCHITECTURE.md](ARCHITECTURE.md) — program layers  
