# Duke's Bank Demo

**Reference application** for Anchor Migration Phase 2 — not a dedicated product repo.

`java-ast-ssot` is a **generic Java AST exporter**; Duke's Bank validates the first stack profile **`javaee-ejb2-jboss`** (EJB 2.x + JBoss CMP XML) plus schema crosswalk. See [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md).

| Section | Status |
|---------|--------|
| Architecture & design decisions | **Design** — locked |
| Database SSOT runbook | **Verified** — Docker MySQL 5.7, export + verify pass (2026-06-27) |
| Java DRG / XML SSOT | **Alpha** — `java-ast-ssot` export verified on bank module (2026-06-27) |
| Crosswalk + linked SSOT | **Verified** — 32 links, 0 errors (2026-06-27) |
| Anchor Explorer UI | **Verified** — load `dukesbank-linked.db` (2026-06-27) |
| **End-to-end runbook** | **Verified** — see [E2E quick path](#e2e-quick-path); Step 7 LLM context optional |

---

## Purpose

Duke's Bank is the **canonical end-to-end sample** for Anchor Migration:

1. Export **schema SSOT** from a real MySQL database (`db-metadata`).
2. Export **Java AST SSOT** from sources (`java-ast-ssot` **core**) plus **Java EE profile** output from EJB/JBoss XML.
3. Link the two graphs (entity bean ↔ table ↔ column).
4. Later: apply OpenRewrite recipes and parity verification.

Why Duke's Bank:

- Classic **J2EE 1.4 / EJB 2.x CMP** — no JPA; mapping lives in XML as well as Java.
- This fork is already configured for **MySQL** (not Derby-only tutorial defaults).
- Small enough to understand, rich enough for FK-less join tables, session beans, and web tier.

---

## Repository layout (local)

Duke's Bank is **not** in the Anchor Migration org — we have **not** forked it yet. Maintainers and contributors use the same layout: an **external** clone ([jiananwang/dukesbank](https://github.com/jiananwang/dukesbank)) as a **sibling** of `anchor-migration` under one parent directory.

```
github/                          e.g. C:\github\
├── anchor-migration/
│   ├── migration-hub/       # this documentation
│   ├── db-metadata/         # schema export CLI
│   ├── demo-dukesbank/      # Docker MySQL bridge + E2E scripts (verified)
│   ├── java-ast-ssot/       # Java AST SSOT + crosswalk (alpha)
│   └── anchor-explorer/     # Read-only crosswalk UI (alpha)
└── dukesbank/               # external clone — NOT inside anchor-migration
    └── data/mysql/dukesbank.sql
```

**Author machine:** `C:\github\dukesbank` next to `C:\github\anchor-migration`  
**Compose mount from `demo-dukesbank/`:** `../../dukesbank/data/mysql/dukesbank.sql`  
**Bank module root (java-ast-ssot):** `dukesbank/src/j2eetutorial14/examples/bank/`

---

## Duke's Bank technical profile

| Aspect | Value |
|--------|-------|
| Era | J2EE 1.4, EJB 2.x **container-managed persistence (CMP)** |
| Persistence | `ejb-jar.xml`, `jbosscmp-jdbc.xml` — **not JPA** |
| App server (original) | JBoss 4.x |
| Database (this fork) | **MySQL** — `jdbc:mysql://…/dukesbank` |
| Java language level | Source **1.4** (Ant `build.properties`) |
| Build | Ant-primary; optional Maven wrapper for RPM |

### Database tables

From `dukesbank/data/mysql/dukesbank.sql`:

| Table | Role |
|-------|------|
| `ACCOUNT` | Bank accounts |
| `CUSTOMER` | Customers |
| `TX` | Transactions |
| `CUSTOMER_ACCOUNT_XREF` | Customer ↔ account many-to-many link |
| `NEXT_ID` | ID sequence counters per bean name |

**Note:** The MySQL seed script defines **primary keys only**. Referential integrity is enforced in the **EJB layer** and via the xref table, not declared as SQL `FOREIGN KEY` constraints. Schema SSOT alone will not show FK edges for most relationships — the **code/XML crosswalk** is required.

### Key Java packages (bank module)

| Package | Contents |
|---------|----------|
| `com.sun.ebank.ejb.account` | `AccountBean` (CMP entity), `AccountControllerBean` (stateful session) |
| `com.sun.ebank.ejb.customer` | `CustomerBean`, `CustomerControllerBean` |
| `com.sun.ebank.ejb.tx` | `TxBean`, `TxControllerBean` |
| `com.sun.ebank.ejb.util` | `NextIdBean` |
| `com.sun.ebank.web` | Web tier beans / servlets |
| `com.sun.ebank.util` | DTOs (`AccountDetails`, …) |
| `com.jboss.ebank` | `TellerBean` (web services module) |

### Deployment descriptors (critical for DRG)

| File | Purpose |
|------|---------|
| `SOURCE_ROOT/dd/ejb/ejb-jar.xml` | EJB names, CMP fields, relationships, EJB-QL |
| `SOURCE_ROOT/dd/ejb/jbosscmp-jdbc.xml` | **EJB ↔ table/column** mapping, MySQL dialect |
| `SOURCE_ROOT/dd/ejb/jboss.xml` | JNDI names, datasource binding |
| `examples/example1/jboss/.../mysql-ds.xml` | JDBC URL, credentials |

`SOURCE_ROOT` = `src/j2eetutorial14/examples/bank/`

---

## Phase 2 architecture: dual SSOT + DRG

```mermaid
flowchart TB
  subgraph legacy [Duke's Bank Legacy]
    MySQL[(MySQL dukesbank DB)]
    Java[Java sources .java]
    XML[EJB / JBoss XML]
  end

  subgraph extract [Deterministic extractors]
    DBMeta[db-metadata CLI]
    CodeExt[java-ast-ssot]
    XMLExt[XML in java-ast-ssot]
  end

  subgraph ssot [SSOT snapshots]
    SchemaDB[(schema SSOT SQLite)]
    JavaDB[(Java AST SSOT SQLite)]
  end

  subgraph link [Crosswalk]
    XWalk[code_schema_link]
  end

  MySQL --> DBMeta --> SchemaDB
  Java --> CodeExt --> JavaDB
  XML --> XMLExt --> JavaDB
  SchemaDB --> XWalk
  JavaDB --> XWalk
```

**DRG (dependency / reference graph)** in this demo means:

- Types, methods, fields, imports, call edges (from Java)
- EJB names, CMP fields, relationships (from XML + Java)
- Table/column bindings (from `jbosscmp-jdbc.xml`)
- **Crosswalk edges:** `AccountBean` ↔ `ACCOUNT` ↔ `db_table` row in schema SSOT

---

## Design decision: AST vs LST vs XML

> **Formal decision record:** [ADR-003 — AST core + sidecars vs LST rewrite layer](ADR-003-ast-sidecar-vs-lst-rewrite-layer.md)

### Three inputs, three roles

| Source | Tool | Role in SSOT |
|--------|------|--------------|
| **XML descriptors** | Deterministic XML parser (XPath/DOM) | **P0** — authoritative for EJB↔table mapping in Duke's Bank |
| **Java structure** | **JavaParser** (AST) | **P0** — types, methods, calls, imports, implements/extends |
| **Java source (lossless)** | **OpenRewrite LST** | **Transform-time only** — recipe development and codemods, not long-term SSOT storage |

### Why not LST for SSOT?

OpenRewrite's **Lossless Semantic Tree** preserves every byte (including comments) via **prefix/suffix** on tree nodes. That makes it ideal for **rewriting** source, but poor as a normalized DRG store:

- Comments sit in prefix/suffix, not on semantic nodes.
- A line comment may apply to the next statement; a block comment may apply to an entire block — **there is no stable, universal comment→code mapping**.
- Duke's Bank uses **Java 1.4** syntax; OpenRewrite targets newer Java levels for parsing.

**Policy:** SSOT stores **verifiable structure**. Comments are a **separate optional layer** (see below).

### Comment handling (v1)

Do **not** attach comments to AST statement nodes in v1.

Instead, store raw comment blocks:

```
source_comment(file, start_line, end_line, kind, text)
```

Optional v2 heuristics (Javadoc before declaration, `//` on previous line) with `confidence = heuristic` — never used on the parity critical path.

This matches OpenRewrite practice: comments travel with the token stream, not with semantic ownership.

### Summary table

| Question | Answer |
|----------|--------|
| What extracts the DRG? | JavaParser + XML parser |
| What rewrites code later? | OpenRewrite LST |
| Where do comments go? | Separate table, weakly linked |
| What links code to DB? | `jbosscmp-jdbc.xml` crosswalk + schema SSOT |

---

## Crosswalk example: AccountBean ↔ ACCOUNT

> **Formal contract:** [ADR-004 — mapping roles and edge kinds](ADR-004-crosswalk-contract-mapping-roles-and-edge-kinds.md). Duke's Bank entity beans are **`persistent_entity`** (`type_maps_to_table` + `field_maps_to_column` after link).

### XML (`jbosscmp-jdbc.xml`)

```xml
<entity>
    <ejb-name>AccountBean</ejb-name>
    <table-name>ACCOUNT</table-name>
    <cmp-field>
        <field-name>accountId</field-name>
        <column-name>account_id</column-name>
    </cmp-field>
    ...
</entity>
```

### Java (`AccountBean.java`)

Abstract CMP entity bean — fields accessed via container; implementation class maps to `AccountBean` in `ejb-jar.xml`.

### Schema SSOT (`db-metadata` export)

Expected stable IDs (MySQL database name = schema in MySQL dialect):

| Entity | Stable ID |
|--------|-----------|
| Table | `dukesbank.ACCOUNT` |
| Column | `dukesbank.ACCOUNT.ACCOUNT_ID` (physical column name from DB) |

**Column name case:** MySQL on Linux may expose uppercase table/column names as defined in `dukesbank.sql`. Exporter records names as returned by the live catalog — always reconcile with `db-metadata verify`.

### Planned crosswalk edge (normalized, post-link)

```
mapping_role: persistent_entity

(com.sun.ebank.ejb.account.AccountBean)
  --type_maps_to_table--> (dukesbank.ACCOUNT)
  --field_maps_to_column--> (accountId → dukesbank.ACCOUNT.ACCOUNT_ID)
```

Profile export also records intermediate edges (`java_type_to_ejb`, `ejb_to_table`) before normalization — see ADR-004.

---

## Phase A: Database SSOT runbook

**Status: Verified (2026-06-27)** — Docker compose in `demo-dukesbank`. MySQL 5.7 on port 3306.

### Prerequisites

- Docker Desktop
- `db-metadata` installed: `pip install -e ".[mysql]"`
- Duke's Bank cloned at `../../dukesbank` relative to `demo-dukesbank` (sibling of `anchor-migration`)

### A.1 Start MySQL

Planned `docker-compose.yml` (MySQL 5.7 for legacy driver compatibility):

```yaml
services:
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: dukesbank
      MYSQL_USER: dukesbank
      MYSQL_PASSWORD: dukesbank
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3306:3306"
    volumes:
      - ../../dukesbank/data/mysql/dukesbank.sql:/docker-entrypoint-initdb.d/01-dukesbank.sql:ro
```

```bash
cd demo-dukesbank
docker compose up -d
# wait for healthy container
```

### A.2 Export schema SSOT

```bash
cd db-metadata
db-migration export \
  --url "mysql+pymysql://dukesbank:dukesbank@localhost:3306/dukesbank" \
  --out metadata/dukesbank.db
```

### A.3 Verify export against live database

```bash
db-migration verify metadata/dukesbank.db \
  --url "mysql+pymysql://dukesbank:dukesbank@localhost:3306/dukesbank"
```

Expected (verified 2026-06-27 on MySQL 5.7):

| Metric | Expected |
|--------|----------|
| Tables | 5 |
| Columns | 27 |
| SQL FOREIGN KEY constraints | 0 |
| Primary keys | 4 |
| Indexes (non-PK) | 0 |
| Matched entities (verify) | 36 |

### A.4 Verification checklist

- [x] Docker MySQL starts cleanly on port 3306
- [x] `export` completes without error
- [x] `verify` exits 0
- [x] `db-migration info metadata/dukesbank.db` shows 5 tables
- [x] Table names match uppercase `ACCOUNT`, `CUSTOMER`, …
- [x] Column count documented: **27** (not 35)

---

## Phase B: Java DRG runbook (profile `javaee-ejb2-jboss`)

**Owner repo:** `java-ast-ssot` — **Alpha verified on Duke's Bank (2026-06-27)**

Uses **core** JavaParser export plus **Java EE EJB 2.x / JBoss CMP** descriptor parsing. v0.1 runs the profile implicitly when XML files are present; refactor will add explicit `--profile`.

### B.1 Scope v1

| Input path | Parser | SSOT entities |
|------------|--------|---------------|
| `examples/bank/src/**/*.java` | JavaParser (language level 1.4) | compilation_unit, type, method, field, call, import |
| `examples/bank/dd/ejb/*.xml` | XML parser | ejb_entity, cmp_field, table_binding, column_binding |
| `examples/bank/dd/ws/*.xml` | XML parser | web service bindings (P1) |

### B.2 Out of scope v1

- Full bytecode / classpath resolution
- JSP parsing
- Comment-to-code semantic binding
- OpenRewrite LST export (defer to `rewrite-recipes` dev workflow)

### B.3 CLI

```bash
# Build (requires JDK 17+; or use Docker Maven image)
cd java-ast-ssot
mvn package

java -jar target/java-ast-ssot-1.0.0-SNAPSHOT.jar export \
  --source-root /path/to/dukesbank/src/j2eetutorial14/examples/bank \
  --profile javaee-ejb2-jboss \
  --out metadata/dukesbank-code.db

java -jar target/java-ast-ssot-1.0.0-SNAPSHOT.jar info \
  --db metadata/dukesbank-code.db
```

**Verified snapshot (2026-06-27):**

| Metric | Value |
|--------|-------|
| Java files | 61 |
| Java types | 61 |
| Java methods | 406 |
| EJB beans | 8 (4 entity CMP + 4 session) |
| Crosswalk edges | 8 (4 `java_type_to_ejb` + 4 `ejb_to_table`) |

Planned next → **implemented** (`crosswalk` CLI, ADR-004 Step 2):

```bash
java -jar target/java-ast-ssot-1.0.0-SNAPSHOT.jar crosswalk \
  --code-db metadata/dukesbank-code.db \
  --schema-db metadata/dukesbank.db \
  --db-schema dukesbank \
  -o metadata/dukesbank-linked.db
```

Exit code `1` when schema targets are missing (`--fail-on-error`, default). Linked DB tables: `crosswalk_run`, `code_schema_link`, `crosswalk_issue`.

### B.4 Verification checklist

- [x] All 4 CMP entity beans discovered (`AccountBean`, `CustomerBean`, `TxBean`, `NextIdBean`)
- [x] `AccountBean` linked to `ACCOUNT` via `jbosscmp-jdbc.xml`
- [x] Session beans listed from `ejb-jar.xml`
- [ ] Call edges from controllers to entity interfaces (P1)

---

## Phase C: Crosswalk + Anchor Explorer

**Status: Verified (2026-06-27)** — `crosswalk` CLI produces linked SSOT; [anchor-explorer](https://github.com/anchor-migration/anchor-explorer) visualizes bidirectional edge colors ([ADR-005](ADR-005-multi-tier-alignment-and-ssot-explorer.md)).

### C.1 Crosswalk (link code SSOT ↔ schema SSOT)

Schema SSOT lives in **`db-metadata/metadata/`**; code + linked SSOT in **`java-ast-ssot/metadata/`**. No need to copy files if paths are passed explicitly.

**Linux / macOS (Docker Maven):**

```bash
cd java-ast-ssot
docker run --rm -v "$PWD:/app" -v "../db-metadata:/dbmeta:ro" -w /app \
  maven:3.9-eclipse-temurin-17 \
  java -jar target/java-ast-ssot-1.0.0-SNAPSHOT.jar crosswalk \
  --code-db metadata/dukesbank-code.db \
  --schema-db /dbmeta/metadata/dukesbank.db \
  --db-schema dukesbank \
  -o metadata/dukesbank-linked.db
```

**Windows (PowerShell):**

```powershell
cd C:\github\anchor-migration\java-ast-ssot
docker run --rm `
  -v "C:/github/anchor-migration/java-ast-ssot:/app" `
  -v "C:/github/anchor-migration/db-metadata:/dbmeta:ro" `
  -w /app maven:3.9-eclipse-temurin-17 `
  java -jar target/java-ast-ssot-1.0.0-SNAPSHOT.jar crosswalk `
  --code-db metadata/dukesbank-code.db `
  --schema-db /dbmeta/metadata/dukesbank.db `
  --db-schema dukesbank `
  -o metadata/dukesbank-linked.db
```

**Expected (verified 2026-06-27):**

| Metric | Expected |
|--------|----------|
| Links written | **32** |
| Crosswalk errors | **0** |
| Forward green | **32** |
| `edge_kind` breakdown | 4× `type_maps_to_table`, 4× `stack_bridge`, 24× `field_maps_to_column` |

> `java-ast-ssot info` targets **code** SSOT only. Linked DB has `crosswalk_run` / `code_schema_link` — use Explorer or SQLite queries (see C.2).

### C.2 Anchor Explorer (human review)

```bash
cd anchor-explorer
npm install
npm run dev
```

Open http://127.0.0.1:5173/ and load:

```
java-ast-ssot/metadata/dukesbank-linked.db
```

**Verification checklist:**

- [x] Header shows `dbSchema: dukesbank`, **Links: 32**, **Issues: 0**
- [x] Crosswalk graph: code nodes (left) ↔ schema nodes (right)
- [x] Edge labels show `→green ←green` (or mixed colors when drift exists)
- [x] Link table filterable by `edge_kind`

Production preview: `npm run build && npm run preview`.

### C.3 Verification checklist (Phase C)

- [x] `crosswalk` exits 0 with 32 links
- [x] `crosswalk_issue` count is 0
- [x] Explorer loads linked DB without error
- [x] Stats match CLI alignment summary

---

## E2E quick path

**Goal:** Live MySQL → schema SSOT → code SSOT → linked SSOT → Explorer — **~30 minutes** first time, **~10 minutes** with warm Docker cache.

**Prerequisites:** Docker Desktop; Duke's Bank at `../../dukesbank` (sibling of `anchor-migration`); `db-metadata` installed (`pip install -e ".[mysql]"`); Node.js 20+ for Explorer; optional local JDK 17+ or Docker Maven only.

### Step 0 — Layout

```
github/
├── anchor-migration/    (db-metadata, demo-dukesbank, java-ast-ssot, anchor-explorer)
└── dukesbank/           (external clone)
```

### Step 1 — MySQL (Phase A)

```bash
cd demo-dukesbank
docker compose up -d
# wait until healthy
```

Or run the helper script (Windows):

```powershell
cd demo-dukesbank
.\scripts\run-e2e.ps1
```

### Step 2 — Schema SSOT

```bash
cd db-metadata
db-migration export \
  --url "mysql+pymysql://dukesbank:dukesbank@localhost:3306/dukesbank" \
  --out metadata/dukesbank.db
db-migration verify metadata/dukesbank.db \
  --url "mysql+pymysql://dukesbank:dukesbank@localhost:3306/dukesbank"
```

### Step 3 — Code SSOT (Phase B)

Build + export with Docker Maven (mount bank source read-only):

```bash
cd java-ast-ssot
docker run --rm -v "$PWD:/app" -w /app maven:3.9-eclipse-temurin-17 mvn -B -q package -DskipTests

docker run --rm \
  -v "$PWD:/app" \
  -v "/path/to/dukesbank/src/j2eetutorial14/examples/bank:/bank:ro" \
  -w /app maven:3.9-eclipse-temurin-17 \
  java -jar target/java-ast-ssot-1.0.0-SNAPSHOT.jar export \
  -s /bank --profile javaee-ejb2-jboss -o metadata/dukesbank-code.db
```

Windows bank mount: `C:/github/dukesbank/src/j2eetutorial14/examples/bank:/bank:ro`

### Step 4 — Crosswalk (Phase C.1)

See [Phase C](#phase-c-crosswalk--anchor-explorer) — schema DB from `db-metadata`, output `java-ast-ssot/metadata/dukesbank-linked.db`.

### Step 5 — Explorer (Phase C.2)

Load `dukesbank-linked.db` in anchor-explorer dev server.

### E2E artifact map

| File | Repo | Role |
|------|------|------|
| `metadata/dukesbank.db` | `db-metadata` | Schema SSOT |
| `metadata/dukesbank-code.db` | `java-ast-ssot` | Code + profile SSOT |
| `metadata/dukesbank-linked.db` | `java-ast-ssot` | Crosswalk + alignment colors |
| `metadata/symbols.db` | `anchor-stubborn` | SCIP symbol graph (Step 7; gitignored) |
| `metadata/account-controller.stub.java` | `anchor-stubborn` | LLM context for session facade (Step 7) |

All `metadata/*.db` files are gitignored — regenerate via this runbook.

### E2E verification summary (2026-06-27)

| Stage | Command / tool | Result |
|-------|----------------|--------|
| MySQL | `docker compose ps` | healthy |
| Schema | `db-migration verify` | 36 matched, exit 0 |
| Code | `export --profile javaee-ejb2-jboss` | 61 types, 8 EJB entities |
| Crosswalk | `crosswalk … -o dukesbank-linked.db` | 32 links, 0 errors |
| Explorer | load linked.db | graph + table render |
| JPA E2E | `run-e2e-jpa-parity.ps1` | 4 entities + NextId; per-entity parity gates, exit 0 |
| LLM context (optional) | `run-stubborn-context.ps1` / `dukesbank-e2e` | AccountControllerBean neighbors + compression KPI |

### Step 6 — JPA re-export + parity (ADR-004 Step 4d)

```powershell
cd demo-dukesbank
.\scripts\run-e2e-jpa-parity.ps1
```

Applies CMP→JPA recipes to **AccountBean**, **CustomerBean**, **TxBean**, and **NextIdBean**, re-exports with auto-detected profiles (`javaee-ejb2-jboss` + `jpa`), runs crosswalk before/after, and emits per-entity parity reports under `parity-verify/metadata/` (`dukesbank-parity-accountbean|customerbean|txbean|nextidbean.{json,html}`) using inline multi-entity matrices (`examples/matrices/dukesbank-cmp-jpa-multi-*.yaml`). See [demo-dukesbank README](https://github.com/anchor-migration/demo-dukesbank#jpa-re-export--parity-adr-004-step-4d--adr-007-v04-multi-entity).

### Step 7 — LLM context (`anchor-stubborn`)

**Optional** horizontal step — not in the deterministic SSOT → rewrite → parity path. Use when drafting recipe mappings, scoping a migration target, or reviewing entities without sending full sources to an LLM.

**Prerequisites:** Steps 1–6 unchanged; external `dukesbank` clone; `anchor-stubborn` on PATH or Docker image.

#### One-shot (Windows)

```powershell
cd demo-dukesbank
.\scripts\run-stubborn-context.ps1
```

Or from `anchor-stubborn`:

```powershell
cd anchor-stubborn\examples\dukesbank
.\scripts\run-e2e.ps1
python ..\..\scripts\verify_dukesbank_context.py
```

#### Docker

```bash
cd anchor-stubborn
docker compose build
docker compose run --rm dukesbank-e2e
python scripts/verify_dukesbank_context.py
```

Mount: `dukesbank/src/j2eetutorial14/examples/bank` → `/bank` (sibling of `anchor-migration`).

#### Primary case: `AccountControllerBean`

| Output | Path |
|--------|------|
| Symbol graph | `anchor-stubborn/examples/dukesbank/metadata/symbols.db` |
| Java stub | `.../metadata/account-controller.stub.java` |
| Anchor-DSL | `.../metadata/account-controller.anchor-dsl` |

**Migration scoping (recommended):**

```bash
anchor-stubborn context metadata/symbols.db \
  --target "<AccountControllerBean stable_id>" \
  --format anchor-dsl \
  --member-signatures neighbors \
  --javadoc summary
```

Case: [account-controller-context.md](https://github.com/anchor-migration/anchor-stubborn/blob/main/examples/dukesbank/cases/account-controller-context.md) · [ADR-010](ADR-010-anchor-stubborn-integration.md) · [MCP](https://github.com/anchor-migration/anchor-stubborn/blob/main/docs/MCP.md).

---

## Relationship to other docs

| Document | Relevance |
|----------|-----------|
| [SSOT-SCHEMA.md](SSOT-SCHEMA.md) | Schema stable IDs; future code AST IDs |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Program layers |
| [DEVELOPMENT-MODEL.md](DEVELOPMENT-MODEL.md) | AI-assisted build; deterministic extractors |
| [ROADMAP.md](ROADMAP.md) | Phase 1–2 scheduling |
| [ADR-005](ADR-005-multi-tier-alignment-and-ssot-explorer.md) | Edge coloring + Explorer |
| [demo-dukesbank README](https://github.com/anchor-migration/demo-dukesbank) | Docker bridge + `scripts/run-e2e.ps1` |
| [ADR-010](ADR-010-anchor-stubborn-integration.md) + [dukesbank example](https://github.com/anchor-migration/anchor-stubborn/tree/main/examples/dukesbank) | Step 7 — LLM context |

---

## Open questions

| # | Question | Resolution path |
|---|----------|-----------------|
| 1 | MySQL 5.7 vs 8.0 for demo? | Start 5.7; document 8.0 if tested |
| 2 | Uppercase table names on Windows Docker? | Record in Phase A checklist |
| 3 | JavaParser 1.4 vs ECJ for very old syntax? | Spike during `java-ast-ssot` POC |
| 4 | Store XML SSOT in same SQLite as Java or separate file? | Prefer single `export_run` per snapshot set with `artifact_type` column |

---

## Revision history

| Date | Change |
|------|--------|
| 2026-07-01 | **Step 7** — anchor-stubborn Duke's Bank LLM context runbook (`dukesbank-e2e`, `run-stubborn-context.ps1`) |
| 2026-06-27 | **E2E runbook** — Phase C (crosswalk + anchor-explorer), quick path verified (32 links) |
| 2026-06-27 | Phase A + B verified locally — MySQL 5.7 verify fix, java-ast-ssot export |
| 2026-06-27 | Initial design doc — architecture locked, runbooks planned |
