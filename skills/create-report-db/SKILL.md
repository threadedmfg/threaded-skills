---
name: create-report-db
description: >-
  Inspect raw customer data (XLSX, CSV, JSON, or API), understand their business
  and operations, normalize the data into a well-structured SQLite database, and
  generate the full DB infrastructure: schema SQL, Python loaders, docs, and a
  report-config.yaml stub. The database is designed to support multiple reports,
  not a single view. Use when setting up a new customer's data layer from scratch.
---

# Create Customer Database

> **Before starting:** Read the `REQUIRED-threaded-runner-setup` skill to determine whether you are running via MCP or CLI, check auth status, and confirm the organization UUID.

## Collect inputs before starting

Ask the user for these if not already provided:

- **Source data path(s)** (required) — path to XLSX workbook, CSV file(s), JSON export, or a description of an API endpoint
- **`ORG_UUID`** (required) — Threaded organization UUID for the `report-config.yaml` stub

---

## Phase 1: Understand the business and inspect the data

The goal of this phase is to understand *what this business does* and *what decisions they need their data to answer* — not just what one report should show. Most Threaded customers are manufacturers. The database should model their operations in a way that can support many different reports over time.

### 1a. Detect format and inspect structure

Auto-detect the source format and extract metadata:

**XLSX workbook** (install `openpyxl` with `pip3 install openpyxl` if needed):
- List all worksheet tabs with row counts
- For each tab: column names, inferred column types, 5 sample rows
- Note any tabs that look like metadata, instructions, or pivot tables (likely skip these)

**CSV file(s)**:
- For each file: column names from header row, first 10 data rows
- Detect delimiter (comma, tab, pipe) and encoding
- Note file sizes and row counts

**JSON export**:
- Top-level structure (array, object with nested arrays, etc.)
- 3 example elements from each array
- Key names and inferred types

**API (described)**:
- Ask the user to describe the endpoint, authentication, response shape, and pagination
- Generate a stub loader that the user will complete

### 1b. Ask the user about their business

Present the source structure summary and ask open-ended questions to understand the operational context. This drives schema decisions far more than column names alone:

```
Source: <filename> (<N> tabs/files)
Data detected: <list of tabs/files with row counts>

To design a database that will serve you well across multiple reports, I have a
few questions about your business:

1. What does your company make or do? (product types, manufacturing process,
   industry — e.g. vehicle assembly, pet food production, contract manufacturing)

2. What are the key operational entities in your data? For example:
   - Do you have a build plan or work order list?
   - Do you track parts, BOMs, or raw materials?
   - Are there purchase orders or supplier deliveries?
   - Is sales or order data included?
   - Are employees, planners, or departments tracked?

3. What decisions do you make from this data? (examples: which builds are at risk,
   which parts to reorder, which customers to prioritize, where costs are highest)

4. Are there known data quality issues? (missing values, inconsistent formats,
   unmapped categories, data from multiple systems that don't share the same keys)

5. Is there other data you'd want to connect to this later, even if it's not in
   the source files today?
```

Wait for the user's response before proceeding to Phase 2.

### 1c. Identify entities and relationships

From the source data and the user's answers, map the business to data entities:

**Common manufacturing entities to look for:**

| Business concept | Likely table | Key relationships |
|---|---|---|
| Build / work order / job | `builds` | → products, parts, employees |
| Part / material / SKU | `parts` | → suppliers, BOMs |
| Bill of materials | `boms` | → parts, builds/products |
| Purchase order | `purchase_orders` | → suppliers, parts |
| PO line item | `po_line_items` | → purchase_orders, parts |
| Supplier / vendor | `suppliers` | ← parts, PO line items |
| Employee / planner | `employees` | ← parts, builds |
| Product / model | `products` | ← builds, BOMs |
| Order / sale | `orders` | → customers, products |
| Order line item | `order_line_items` | → orders, products |
| Customer / account | `customers` | ← orders |
| Inventory snapshot | `inventory` | → parts |
| Quality event | `quality_events` | → builds, parts |

Not every customer will have all of these. Identify which entities exist in their data and propose a schema that covers what's present and anticipates what's likely to be needed.

---

## Phase 2: Propose schema

### 2a. Design the normalized schema

Based on the source structure and user input, propose:

- **Table names** — lowercase, snake_case, plural (e.g., `builds`, `parts`, `suppliers`)
- **Columns** — name, SQLite type (`TEXT`, `INTEGER`, `REAL`), nullable, constraints
- **Primary keys** — natural keys preferred (e.g., `part_num`, `build_id`, `po_number`); auto-increment `_id` only if no natural key exists
- **Foreign keys** — with `REFERENCES` clauses; note `ON DELETE` behavior where relevant
- **Indexes** — on foreign keys, date columns, and any column likely to be filtered in WHERE clauses

Design for the full operational picture, not just what can be seen in the source today. If the user mentioned they want to connect PO data later, leave room for it in the schema (a `supplier_id` FK on `parts` even if the suppliers table is stubbed).

### 2b. Design analytical views

Propose views that compute common aggregations and business logic. These are general-purpose — not tied to a specific report — and serve as the queryable layer above the raw tables:

**Common operational views for manufacturers:**
- `v_effective_schedule` — resolves actual vs. planned dates for builds/jobs
- `v_runout_dates` — first date projected inventory reaches zero per part
- `projected_available_parts` — running inventory balance by part/date (if BOM + PO data available)
- `v_at_risk_builds` — builds where a required part runs out before the start date
- `v_revenue_by_period` — monthly/quarterly sales aggregations
- `v_supplier_exposure` — parts per supplier, line-stopper count, earliest runout

Propose views that make sense given the data. Don't generate views for data that isn't present.

### 2c. Present for confirmation

Show the full schema and explain the design choices. This is a single confirmation — present tables and views together:

```
Proposed Schema for <customer_name>
════════════════════════════════════

This database models your <build plan / sales + COGS / production schedule> data
and is designed to support multiple reports over time.

Tables:
  parts (from tab "Inventory")
    part_num        TEXT    PK     natural key from source
    part_name       TEXT    NOT NULL
    supplier_id     TEXT    FK → suppliers.supplier_id
    line_stopper    INTEGER        0/1 flag
    qty_on_hand     REAL
    material_planner_id  TEXT  FK → employees.employee_id

  suppliers (from tab "Inventory" / supplier column)
    supplier_id     TEXT    PK     normalized from parts.supplier
    supplier        TEXT    NOT NULL

  builds (from tab "Build Plan")
    build_id        TEXT    PK
    product_id      TEXT    FK → products.product_id
    planned_start_date   TEXT
    actual_start_date    TEXT
    planned_complete_date TEXT
    actual_complete_date  TEXT
    parts_ready     INTEGER        0/1, recomputed on each load

  <... additional tables ...>

Views:
  v_runout_dates
    First projected stockout date per part, joined to planner
    Supports: Part Runout report, Build Status risk computation

  projected_available_parts
    Daily running balance: qty_on_hand + PO receipts − BOM demand
    Supports: Projected Inventory report, v_runout_dates

Design notes:
  - parts_ready is computed at load time, not stored in the source
  - <any other non-obvious decisions>

Reply with any changes, or "ok" to proceed.
```

**Wait for the user to approve before generating any files.**

---

## Phase 3: Generate artifacts

After the user approves the schema, generate these files in a `db/` directory (or a location the user specifies):

### 3a. Schema SQL

Create a `schema/` directory with one `.sql` file per table or view. The filename matches the object name exactly (e.g., `schema/parts.sql`, `schema/v_runout_dates.sql`). Each file contains only the DDL for that object — no comments, no extra statements.

**Table file example (`schema/parts.sql`):**

```sql
CREATE TABLE IF NOT EXISTS parts (
    part_num            TEXT PRIMARY KEY,
    part_name           TEXT NOT NULL,
    supplier_id         TEXT REFERENCES suppliers(supplier_id),
    line_stopper        INTEGER DEFAULT 0,
    qty_on_hand         REAL
);

CREATE INDEX IF NOT EXISTS idx_parts_supplier_id
    ON parts (supplier_id);
```

**View file example (`schema/v_runout_dates.sql`):**

```sql
CREATE VIEW IF NOT EXISTS v_runout_dates AS
SELECT
    p.part_num        AS part_number,
    MIN(pap.dt)       AS runout_date,
    e.name            AS material_planner
FROM projected_available_parts pap
JOIN parts p ON p.part_num = pap.part_number
LEFT JOIN employees e ON e.employee_id = p.material_planner_id
WHERE pap.running_balance <= 0
GROUP BY pap.part_number;
```

`create_db.py` applies all files in `schema/` in two passes: tables first (all files not starting with `v_`), then views (files starting with `v_`). This ensures referenced tables exist before any view that joins them. Within each pass, apply in dependency order if views reference other views.

### 3b. Database creation script

**`create_db.py`** — creates the SQLite database and applies schema:

```python
# Usage: python3 create_db.py [--reset]
# --reset drops all existing tables and views, then recreates from schema/
```

Must support `--reset` to drop and recreate everything. Apply `tables.sql` then `views.sql` in order.

### 3c. Loader scripts

**`loaders/load_{source}.py`** — one loader per source tab or file. Each loader:

- Reads the source data (XLSX tab, CSV file, JSON array) using the appropriate library
- Transforms and validates rows according to the schema mapping
- Normalizes values that are denormalized in the source (e.g., splits `supplier` column into `suppliers` table rows)
- Inserts into the database using parameterized queries
- Reports row counts on completion

**`loaders/run_all.py`** — orchestrator that:

- Calls each loader in dependency order (referenced tables loaded first)
- Computes any derived/flag columns after all raw data is loaded (e.g., recompute `parts_ready` after both `builds` and `projected_available_parts` are populated)
- Reports total row counts per table
- Runs `PRAGMA foreign_key_check` at the end

### 3d. Summary script

**`summarize.py`** — runs key queries to give a quick operational health check of the data. Output is captured as `{SUMMARY_OUTPUT}` by the `generate-report-insights` skill. Include:

- Row counts per table
- Key business aggregations (e.g., total builds, on-track vs. at-risk, total parts, PO coverage)
- Date ranges (earliest/latest build dates, PO dates, etc.)
- Data quality flags (zero-row tables, parts with no supplier, builds with no BOM entries)

Name this `summarize.py` (not tied to a specific report name) since it summarizes the whole database.

### 3e. Documentation

**`schema/`** — one `.sql` file per table or view, as described in 3a. This directory is the authoritative DDL source — `create_db.py` reads from it, and any future schema changes are made here first.

```
schema/
├── suppliers.sql
├── employees.sql
├── parts.sql
├── builds.sql
├── purchase_orders.sql
├── po_line_items.sql
├── boms.sql
├── v_runout_dates.sql
└── projected_available_parts.sql
```

**`docs/README.md`** — describes the data model:
- Where the data comes from (source files, systems)
- Rebuild commands (`python3 create_db.py --reset && python3 -m loaders.run_all`)
- What each table contains and its grain (one row per what?)
- Known data quality limitations

**`docs/schema.md`** — Mermaid ER diagram covering all tables:

```markdown
## Entity Relationships

​```mermaid
erDiagram
    suppliers ||--o{ parts : supplies
    parts ||--o{ boms : required_in
    products ||--o{ builds : produced_as
    builds ||--o{ boms : uses
    purchase_orders ||--o{ po_line_items : contains
    parts ||--o{ po_line_items : ordered_via
​```
```

---

## Phase 4: Execute and validate

### 4a. Build the database

```bash
python3 create_db.py --reset
PYTHONPATH=. python3 -m loaders.run_all
```

Or if loaders are direct scripts rather than a module:
```bash
python3 create_db.py --reset
python3 loaders/run_all.py
```

### 4b. Validate

Run row count checks on every table and a foreign key integrity check:

```bash
python3 -c "
import sqlite3
conn = sqlite3.connect('<db_path>')

objects = conn.execute(\"SELECT type, name FROM sqlite_master WHERE type IN ('table','view') ORDER BY type, name\").fetchall()
for obj_type, name in objects:
    count = conn.execute(f'SELECT COUNT(*) FROM [{name}]').fetchone()[0]
    status = '✓' if count > 0 else '⚠ EMPTY'
    print(f'{status} {obj_type:5} {name}: {count} rows')

fk = conn.execute('PRAGMA foreign_key_check').fetchall()
print(f'Foreign key check: {\"OK\" if not fk else fk}')
conn.close()
"
```

Flag any zero-row tables and investigate before proceeding. Common causes: wrong tab name, shifted header rows, encoding issues.

### 4c. Report results

```
Database Created
================
Path: <db_path>
Tables: <count>  Views: <count>
Total rows (tables): <sum>
  ✓ table  suppliers: 24
  ✓ table  parts: 312
  ✓ table  builds: 1,563
  ✓ view   v_runout_dates: 99
  ✓ view   projected_available_parts: 147,420
Foreign key check: OK
```

---

## Phase 5: Generate report-config.yaml stub

Generate a `report-config.yaml` pre-filled with the database infrastructure. The `reports` section is left empty with commented examples — reports are added later when the first report is created with the `create-threaded-report` skill:

```yaml
organization_uuid: <ORG_UUID>

# Reports are added here as each one is created.
# Use the create-threaded-report skill to scaffold and register each report.
# The same database can serve multiple reports simultaneously.
reports: []
  # Example entry (filled in by create-threaded-report):
  # - name: Build Status
  #   slug: build-status
  #   uuid: orp-xxxxxxxx-...
  #   shell: reports/build_status.html
  #   insights_js: reports/build_status.insights.js
  #   insights_prompt: reports/build_status.insights-prompt.md
  #   queries_sql: reports/build_status.sql
  #   db: db/<customer>.db
  #   insight_keys:
  #     - title
  #     - stats

db:
  source_workbook: <source_path>       # or source_data: <dir/> for CSV/JSON
  create_script: create_db.py
  loader_module: loaders.run_all       # or loader_script: loaders/run_all.py
  loader_pythonpath: .
  summarize_script: summarize.py
```

Tell the user the database is ready to use as a foundation for any number of reports, and that each report can add its own views on top without changing the core schema.

---

## Troubleshooting

**`openpyxl` not installed**
Run `pip3 install openpyxl` to install the XLSX reading library.

**Encoding issues with CSV**
Use `encoding="utf-8-sig"` when opening CSVs exported from Excel to strip the invisible BOM character from the first column name.

**Foreign key violations after loading**
Check that loaders run in dependency order — referenced tables must be fully loaded before referencing tables. Verify that source data has consistent key values across related tabs/files (e.g., a part number in the BOM tab that doesn't exist in the parts tab).

**Zero rows in a table after loading**
Common causes: wrong worksheet tab name (case-sensitive in openpyxl), empty rows at the top of the sheet, merged cells that confuse column detection. Inspect the raw data and adjust the loader's row-reading offset.

**Views return zero rows when tables are populated**
Verify that join keys match between tables — a common issue is a whitespace or case mismatch in denormalized keys (e.g., part number with trailing space in one table). Add a `TRIM(LOWER(...))` normalization step in the loader if needed.

**View creation fails**
`create_db.py` applies table files before view files. If a view depends on another view, the dependency must be applied first — rename files with a numeric prefix (e.g., `v_01_daily_consumption.sql`, `v_02_projected_available_parts.sql`, `v_03_runout_dates.sql`) to control the apply order within the views pass.
