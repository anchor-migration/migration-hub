# Duke's Bank Demo

**Reference application** for Anchor Migration Phase 2 — not a dedicated product repo.

`java-ast-ssot` is a **generic Java AST exporter**; Duke's Bank validates the first stack profile **`javaee-ejb2-jboss`** (EJB 2.x + JBoss CMP XML) plus schema crosswalk. See [ADR-002](ADR-002-java-ast-ssot-core-and-profiles.md).

| Section | Status |
|---------|--------|
| Architecture & design decisions | **Design** — locked |
| Database SSOT runbook | **Verified** — Docker MySQL 5.7, export + verify pass (2026-06-27) |
| Java DRG / XML SSOT | **Alpha** — `java-ast-ssot` export verified on bank module (2026-06-27) |
| Crosswalk examples | **Verified** — 4 entity beans linked to tables via XML |

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
│   ├── demo-dukesbank/      # Docker MySQL bridge (verified)
│   └── java-ast-ssot/       # Java AST SSOT (alpha)
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

  subgraph link [Crosswalk planned]
    XWalk[entity to table edges]
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

### Planned crosswalk edge

```
(java_type: com.sun.ebank.ejb.account.AccountBean)
  --maps_to_ejb--> (ejb_name: AccountBean)
  --maps_to_table--> (schema_table: dukesbank.ACCOUNT)
  --field_maps_to_column--> (accountId → ACCOUNT_ID)
```

Each edge gets a deterministic `edge_id` in `java-ast-ssot` and references pinned `export_run_id` from both SSOT files.

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

java -jar target/java-ast-ssot-0.2.0-SNAPSHOT.jar export \
  --source-root /path/to/dukesbank/src/j2eetutorial14/examples/bank \
  --profile javaee-ejb2-jboss \
  --out metadata/dukesbank-code.db

java -jar target/java-ast-ssot-0.1.0-SNAPSHOT.jar info \
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

Planned next:

```bash
java-ast-ssot crosswalk \
  --code-db metadata/dukesbank-code.db \
  --schema-db metadata/dukesbank.db \
  --out metadata/dukesbank-linked.db
```

### B.4 Verification checklist

- [x] All 4 CMP entity beans discovered (`AccountBean`, `CustomerBean`, `TxBean`, `NextIdBean`)
- [x] `AccountBean` linked to `ACCOUNT` via `jbosscmp-jdbc.xml`
- [x] Session beans listed from `ejb-jar.xml`
- [ ] Call edges from controllers to entity interfaces (P1)

---

## Relationship to other docs

| Document | Relevance |
|----------|-----------|
| [SSOT-SCHEMA.md](SSOT-SCHEMA.md) | Schema stable IDs; future code AST IDs |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Program layers |
| [DEVELOPMENT-MODEL.md](DEVELOPMENT-MODEL.md) | AI-assisted build; deterministic extractors |
| [ROADMAP.md](ROADMAP.md) | Phase 1–2 scheduling |

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
| 2026-06-27 | Phase A + B verified locally — MySQL 5.7 verify fix, java-ast-ssot export |
| 2026-06-27 | Initial design doc — architecture locked, runbooks planned |
