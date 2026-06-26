# Anchor Migration

**SSOT-driven toolkit for AI-assisted legacy modernization.**

Anchor Migration treats **database schema** and **source code AST** as single sources of truth (SSoT), applies **OpenRewrite** recipes for mechanical refactoring, and uses **AST + AI** to verify business parity between old and new code.

This program is a personal showcase of **architecture-led, AI-assisted engineering** applied to legacy modernization: the developer defines structure and boundary protocols; AI implements most of the code; the shipped artifacts are **deterministic Python and Java** with verification gates at every stage.

> [Development model →](docs/DEVELOPMENT-MODEL.md) — roles, boundary protocols, deterministic core vs optional AI self-healing nodes.

> **New here?** Read **[START-HERE.md](docs/START-HERE.md)** for the full program map. AI assistants: see **[AGENTS.md](AGENTS.md)**.

## Repositories

| Repository | Role | Status |
|------------|------|--------|
| [**migration-hub**](https://github.com/anchor-migration/migration-hub) | Program overview, architecture, roadmap | Active |
| [**db-metadata**](https://github.com/anchor-migration/db-metadata) | Live DB → schema SSOT (SQLite) | Alpha |
| [**java-ast-ssot**](https://github.com/anchor-migration/java-ast-ssot) | Java source → Java AST SSOT | Alpha |
| [**rewrite-recipes**](https://github.com/anchor-migration/rewrite-recipes) | OpenRewrite rule catalog | Planned |
| [**parity-verify**](https://github.com/anchor-migration/parity-verify) | Old vs new business parity verification | Planned |
| [**pattern-catalog**](https://github.com/anchor-migration/pattern-catalog) | Migration pattern docs and examples | Planned |

## Pipeline

```mermaid
flowchart TB
  subgraph sources [Legacy Sources]
    LiveDB[(Live Database)]
    LegacyCode[(Legacy Java Code)]
  end

  subgraph ssot [SSoT Layer]
    MetaDB[(Schema SSOT)]
    JavaSSOT[(Java AST SSOT)]
  end

  subgraph transform [Transformation]
    OR[OpenRewrite Recipes]
    AI[AI-assisted Refactoring]
  end

  subgraph verify [Verification]
    ASTDiff[AST / Semantic Diff]
    Parity[Parity Tests]
  end

  subgraph output [Target]
    NewCode[(Modernized Code)]
  end

  LiveDB -->|db-metadata| MetaDB
  LegacyCode -->|java-ast-ssot| JavaSSOT
  MetaDB --> OR
  JavaSSOT --> OR
  JavaSSOT --> AI
  OR --> NewCode
  AI --> NewCode
  LegacyCode --> ASTDiff
  NewCode --> ASTDiff
  MetaDB --> Parity
  ASTDiff --> Parity
```

## Design principles

1. **SSoT first** — Schema and AST are exported from live systems, not inferred from docs.
2. **Deterministic core** — Shipped tools are replayable Python/Java; same inputs → same outputs.
3. **Architecture-led, AI-implemented** — Developer owns design and boundary contracts; AI writes most code against them.
4. **Mechanical where possible** — OpenRewrite and codemods for repeatable patterns.
5. **AI where necessary** — Ambiguous patterns, test generation, parity exploration — always gated by verify.
6. **Verify always** — Every migration path must be checkable against ground truth.
7. **Composable tools** — Small repos, clear contracts (SQLite / JSON schemas), independent release cycles.

Optional **AI self-healing nodes** (suggestions, test exploration) may sit beside the pipeline but never replace deterministic export, transform, or verify steps. See [Development model](docs/DEVELOPMENT-MODEL.md).

## Local workspace layout

```
anchor-migration/
├── migration-hub/       # this repository
├── db-metadata/         # Python CLI — schema export
├── demo-dukesbank/      # Duke's Bank MySQL Docker (verified)
├── java-ast-ssot/       # Java AST SSOT exporter (alpha)
├── rewrite-recipes/     # (planned)
├── parity-verify/       # (planned)
└── pattern-catalog/     # (planned)
```

Open `anchor-migration.code-workspace` in Cursor/VS Code to work across repos.

## Documentation

- **[Start here](docs/START-HERE.md)** — program map, reading order, conventions
- [AGENTS.md](AGENTS.md) — AI session bootstrap
- [Development model](docs/DEVELOPMENT-MODEL.md) — AI-assisted workflow, boundary protocols, deterministic core
- [Duke's Bank demo](docs/DUKESBANK-DEMO.md) — reference app, DRG design, AST/LST/XML decisions
- [Architecture](docs/ARCHITECTURE.md)
- [Roadmap](docs/ROADMAP.md)
- [SSoT schema contracts](docs/SSOT-SCHEMA.md)

## Getting started

The first available tool is **db-metadata**:

```bash
cd db-metadata
pip install -e ".[all]"
db-migration export --url "..." --out metadata.db
db-migration verify metadata.db --url "..."
```

See [db-metadata README](https://github.com/anchor-migration/db-metadata).

## Contributing

Contributions welcome as the program grows. Start with [ROADMAP.md](docs/ROADMAP.md) for planned work and open issues in each repository.

## License

MIT — see LICENSE in each repository.
