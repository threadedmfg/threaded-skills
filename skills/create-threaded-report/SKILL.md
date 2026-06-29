---
name: create-threaded-report
description: >-
  Scaffold an HTML report shell from a template, create the report in Threaded
  via CLI, and push the initial shell and data revisions. Use when building a
  new report for a customer org that already has a database.
---

# Create Threaded Report

> **Before starting:** Read the `REQUIRED-threaded-runner-setup` skill to determine whether you are running via MCP or CLI, check auth status, and confirm the organization UUID.

## Collect inputs before starting

Ask the user for these if not already provided:

- **`--config`** (optional) — path to existing `report-config.yaml`. If provided, the org UUID and database path are read from it. If omitted, ask for them directly.
- **Report name** (required) — human-readable title (e.g., "Sales by Product")
- **Report slug** (required) — URL-safe identifier (e.g., `sales-by-product`)
- **Report description** (optional) — short description for the report listing

---

## Phase 1: Gather requirements

### 1a. Understand the data

If a `report-config.yaml` exists, read the `db` section to understand the database structure. Query the database to list available tables and views:

```bash
python3 -c "
import sqlite3
conn = sqlite3.connect('<db_path>')
for row in conn.execute(\"SELECT type, name FROM sqlite_master WHERE type IN ('table','view') ORDER BY type, name\"):
    count = conn.execute(f'SELECT COUNT(*) FROM [{row[1]}]').fetchone()[0]
    print(f'{row[0]:5} {row[1]}: {count} rows')
conn.close()
"
```

### 1b. Define the report layout

Ask the user to choose which components the report should include:

**Chart types:**

| Type | Use case |
|---|---|
| Stat chips row | KPI summary numbers at the top |
| Bar chart | Categorical comparisons |
| Line chart | Time series trends |
| Stacked bar | Composition over time |
| Doughnut/pie | Proportional breakdown |
| Gantt chart | Timeline/schedule visualization |
| Heatmap matrix | Two-dimensional density (parts × dates) |
| Data table | Sortable, filterable detail rows |

**Filter controls:**

| Control | Use case |
|---|---|
| Date range buttons | Quick time window selection (30D, 90D, YTD, All) |
| Dropdown select | Filter by category (channel, status, supplier) |
| Search input | Free-text search across table rows |
| Toggle | Binary filter (show/hide a subset) |

### 1c. Confirm the plan

Present a summary of what will be generated:

```
Report: <name> (<slug>)
Organization: <org_uuid>
Database: <db_path>

Layout:
  - Stat chips: Total Revenue, Orders, Products, AOV
  - Bar chart: Revenue by Product (Pareto)
  - Line chart: Monthly Revenue Trend
  - Data table: Product detail with search and sort
  - Filters: Date range, Channel dropdown, Search

Insight keys: title, stats, pareto, monthly, table

Proceed? (reply with changes or "ok")
```

Wait for confirmation before generating files.

---

## Phase 2: Extend the database for this report

Each report may need views, computed columns, or derived tables that are specific to what it shows — without altering the core schema that other reports rely on. This phase adds those report-specific additions to the database before any HTML is generated.

### 2a. Identify what queries the report needs

Based on the layout confirmed in Phase 1, think through what each chart section will query:

- What is the **grain** of the main table? (one row per build, per part, per order line)
- What **joins** are needed? (parts → suppliers, builds → BOMs, orders → customers)
- What **aggregations** are pre-computable? (monthly totals, running balances, at-risk counts)
- What **business logic** needs to be encoded? (status flags, date math, categorization)

Check whether the existing database already has views that satisfy these needs:

```bash
python3 -c "
import sqlite3
conn = sqlite3.connect('<db_path>')
views = conn.execute(\"SELECT name FROM sqlite_master WHERE type='view'\").fetchall()
for v in views:
    count = conn.execute(f'SELECT COUNT(*) FROM [{v[0]}]').fetchone()[0]
    cols = [d[0] for d in conn.execute(f'SELECT * FROM [{v[0]}] LIMIT 0').description or []]
    print(f'{v[0]} ({count} rows): {cols}')
conn.close()
"
```

### 2b. Propose additions

List only what's missing from the existing database. Common report-specific additions:

| Addition | When to add it |
|---|---|
| New view joining tables the report filters on | Core tables exist but aren't pre-joined |
| Date-spine table | Time-series charts need one row per day |
| Running balance view | Inventory / burn-down charts need cumulative totals |
| Computed flag column | Status categories (e.g., `parts_ready`, `urgency_tier`) that the chart colors on |
| Aggregation view | Chart queries the same GROUP BY in multiple places — put it in a view |
| Lookup / mapping table | User-defined categorization (e.g., vendor → protein, region → territory) |

Present proposed additions:

```
Database additions for <report_name>:

New views (appended to schema/views.sql):
  v_monthly_revenue
    Aggregates orders by month; supports the Monthly Trend line chart.
    Joins: orders → order_line_items → products

  v_product_pareto
    Revenue by product with cumulative % for Pareto bar chart.
    Joins: order_line_items → products, aggregated over full history

No new tables needed — existing schema covers this report's requirements.

Reply with any changes, or "ok" to add these and rebuild.
```

If nothing is needed, say so explicitly and skip to Phase 3.

### 2c. Apply additions and rebuild views

After confirmation, write each new view as its own file in `schema/` (e.g., `schema/v_monthly_revenue.sql`) following the same one-file-per-object convention as the rest of the schema directory. Then re-apply only the new view files — no `--reset` needed, the core tables and their data are preserved:

```bash
python3 -c "
import sqlite3, glob
conn = sqlite3.connect('<db_path>')
# Apply only the new view files
for path in sorted(glob.glob('schema/v_monthly_revenue.sql')):  # replace with actual new files
    with open(path) as f:
        conn.executescript(f.read())
conn.commit()
conn.close()
print('Views updated.')
"
```

Verify the new views return rows:

```bash
python3 -c "
import sqlite3
conn = sqlite3.connect('<db_path>')
for view in ['v_monthly_revenue', 'v_product_pareto']:   # replace with actual view names
    count = conn.execute(f'SELECT COUNT(*) FROM [{view}]').fetchone()[0]
    print(f'{view}: {count} rows')
conn.close()
"
```

### 2d. Update database documentation

If any new views or tables were added in 2c, update the docs to reflect the current state of the schema. Skip this step if Phase 2 found no additions were needed.

**`docs/README.md`** — add an entry for each new view under the relevant section. Describe:
- What the view computes (one sentence)
- What reports or queries use it
- Any performance notes (e.g., heavy CROSS JOIN, date-range sensitive)

**`docs/schema.md`** — add the new view(s) to the Mermaid ER diagram. Views that join multiple tables should show their primary relationships:

```markdown
erDiagram
    parts ||--o{ boms : required_in
    builds ||--o{ boms : uses
    v_monthly_revenue }o--|| orders : aggregates
```

If the existing `docs/schema.md` only covers tables, add a second diagram block for views rather than mixing them with the table ER diagram.

**`schema/<view_name>.sql`** — confirm the new file is already in place from step 2c. The `schema/` directory is the authoritative source; docs should describe what's there, not the other way around.

---

## Phase 3: Scaffold the shell

### 3a. Generate the HTML report

Create a self-contained HTML file with these components:

**Required scripts (CDN with SRI + vendored wasm):**
```html
<!-- sql.js 1.10.3 — SRI-locked, wasm loaded from vendor/ -->
<script
  src="https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.10.3/sql-wasm.js"
  integrity="sha384-8D3Rsfo535FqoC1pHCCQMrNf75UgzyoG/HQm9zOzITRrz3QKzecc2E7JXKGCXoWu"
  crossorigin="anonymous"></script>
<!-- Chart.js 4.5.1 — SRI-locked -->
<script
  src="https://cdn.jsdelivr.net/npm/chart.js@4.5.1/dist/chart.umd.min.js"
  integrity="sha384-jb8JQMbMoBUzgWatfe6COACi2ljcDdZQ2OxczGA3bGNeWe+6DChMTBJemed7ZnvJ"
  crossorigin="anonymous"></script>
```

**Standard loading pattern:**
```javascript
async function main() {
    const params = new URLSearchParams(window.location.search);
    const dataUrl = params.get('data');
    if (!dataUrl) { showError('No data URL'); return; }

    const sqlPromise = initSqlJs({
        // wasm is vendored locally — not covered by script-tag SRI
        locateFile: file => `vendor/${file}`
    });
    const dataPromise = fetch(dataUrl).then(r => r.arrayBuffer());
    const [SQL, buf] = await Promise.all([sqlPromise, dataPromise]);
    const db = new SQL.Database(new Uint8Array(buf));

    // Render all sections
    renderAll(db);

    // Wire up insight buttons
    document.querySelectorAll('.insight-btn').forEach(btn => {
        btn.addEventListener('click', () => openInsight(btn.dataset.insight));
    });
}
```

**Insights infrastructure:**
- Copy CSS, modal HTML, and JS functions from `shared/insights-ui.html` (or the reference report)
- Add `// ==INSIGHTS-BEGIN==` and `// ==INSIGHTS-END==` markers around the `const INSIGHTS = { ... };` declaration
- Add sparkle buttons (`<button class="insight-btn" data-insight="key">✦</button>`) to each section header

**Placeholder sections:**
For each requested chart type, read the corresponding snippet from the `examples/` folder in this skill's directory before generating that section. Use the snippet as the starting point and adapt it to the actual query and column names confirmed in Phase 1:

| Chart type | Example file |
|---|---|
| Stat chips / KPI row | `examples/stat-chips.html` |
| Bar chart | `examples/bar-chart.html` |
| Line chart / time series | `examples/line-chart.html` |
| Sortable data table | `examples/data-table.html` |
| Date range + filter controls | `examples/filters.html` |

Each section should include:
- A container div with a descriptive ID
- A card-title div with the section name and sparkle button
- The render function from the example, with the placeholder query replaced by the real one
- A `<!-- TODO: implement <chart_type> -->` comment only for chart types not covered by an example

### 3b. Copy vendored wasm

The JS libraries load from CDN with SRI verification. Only `sql-wasm.wasm` must be copied locally because it is fetched dynamically at runtime and cannot be covered by a script-tag `integrity` attribute:

```bash
mkdir -p <report_dir>/vendor/
cp <skill_dir>/vendor/sql-wasm.wasm <report_dir>/vendor/sql-wasm.wasm
```

Where `<skill_dir>` is the directory containing this `SKILL.md` file and `<report_dir>` is the directory where the report HTML will be written (e.g., `reports/`).

### 3c. Create companion files

**`{slug}.sql`** — stub SQL queries for each chart section:

```sql
-- Companion queries for <report_name>
-- Run against <db_path> to generate data for insights

-- ── 1. Summary stats ─────────────────────────────────────────────────────────
SELECT COUNT(*) AS total_rows FROM <main_table>;

-- ── 2. <Chart section> ───────────────────────────────────────────────────────
-- TODO: Add query for <chart_type>
```

**`{slug}.insights-prompt.md`** — template with report name and keys pre-filled:

```markdown
# <Report Name> — Insights Prompt

## Business Context
<!-- TODO: Describe who uses this report and for what decisions -->

## Data Sources
<!-- TODO: Describe the database tables and where the data comes from -->

## Per-Key Questions

### `title`
<!-- TODO: What should the executive summary highlight? -->

### `stats`
<!-- TODO: What does each KPI chip show? -->

## Format Requirements
- Use HTML tags only in the body field — no markdown
- Use <strong> for key numbers, <h4> for headings, <ul><li> for lists
- Use <div class="insight-note"> for caveats
- 150-word limit per section
- Include {LOAD_DATE} in the title key

## Data Caveats
<!-- TODO: Known data quality issues or limitations -->

## Template Variables
- {LOAD_DATE}, {SUMMARY_OUTPUT}, {QUERY_RESULTS}
```

**`{slug}.insights.js`** — empty INSIGHTS object:

```javascript
const INSIGHTS = {
  'title': { title: '<Report Name> — Overview', body: '<p>Insights will be generated from data.</p>' },
  'stats': { title: 'Summary Metrics', body: '<p>Insights will be generated from data.</p>' },
  // ... one stub per insight key
};
```

---

## Phase 4: Create report in Threaded and do initial push

### 4a. Create the report

**MCP:**
```
execute_threaded_script(script="threaded task org:report:create --organization <org_uuid> --title '<report_name>' --slug '<slug>' --description '<description>'")
```

**CLI:**
```bash
threaded task org:report:create \
  --organization <org_uuid> \
  --title '<report_name>' \
  --slug '<slug>' \
  --description '<description>'
```

Extract the new report UUID from the response.

### 4b. Update report-config.yaml

If a `report-config.yaml` exists, add the new report entry with the UUID:

```yaml
  - name: <Report Name>
    slug: <slug>
    uuid: <new_report_uuid>
    shell: reports/<slug>.html
    insights_js: reports/<slug>.insights.js
    insights_prompt: reports/<slug>.insights-prompt.md
    queries_sql: reports/<slug>.sql
    db: <db_path>
    insight_keys:
      - title
      - stats
      # ... other keys
```

### 4c. Inject stub insights and push shell

Run `build_insights.py` to inject the stub insights into the HTML:

```bash
python3 <scripts_dir>/build_insights.py --html <shell_path> --insights <insights_js_path>
```

Push the shell revision:

```bash
threaded task org:report:shell-revision \
  --organization <org_uuid> \
  --report <report_uuid> \
  --shell-path <shell_path>
```

### 4d. Push initial data

```bash
threaded task org:report:upload-new-data \
  --organization <org_uuid> \
  --report <report_uuid> \
  --data-path <db_path>
```

### 4e. Print the live URL

```
Report Created
==============
Name: <report_name>
Slug: <slug>
UUID: <report_uuid>
Organization: <org_uuid>

Live URL:
  https://app.threadedmfg.com/api/v2/reporting/organizations/<org_uuid>/reports/<report_uuid>/shell/

Files created:
  - reports/<slug>.html
  - reports/<slug>.sql
  - reports/<slug>.insights-prompt.md
  - reports/<slug>.insights.js

Next steps:
  1. Implement the chart rendering functions in the HTML (replace TODO placeholders)
  2. Write companion SQL queries for insights generation
  3. Fill in the insights-prompt.md with business context
  4. Run generate-report-insights to create real insights
  5. Run update-threaded-report to push the final version
```

---

## Usage examples

```bash
# Create a new report with an existing config
create-threaded-report --config /path/to/report-config.yaml

# Create a report interactively (will ask for org UUID, DB path, etc.)
create-threaded-report
```

---

## Troubleshooting

**"Report slug already exists"**
Each report slug must be unique within an organization. Choose a different slug or check if the report already exists with `threaded task org:reports:list`.

**Shell push fails**
Verify the HTML file is valid and contains the required `// ==INSIGHTS-BEGIN==` and `// ==INSIGHTS-END==` markers. Check that `build_insights.py` ran successfully.

**Data push fails**
Verify the database file exists and is a valid SQLite file. Check that the CLI is authenticated to the correct organization.

**`build_insights.py` not found**
The injection script should be in a `scripts/` directory near the report files. If it doesn't exist yet, you can copy it from an existing report group — the script is generic and works with any report that has the markers.

**Wasm file missing after report creation**
If the browser console shows `Failed to load resource: vendor/sql-wasm.wasm`, the wasm file was not copied next to the report. Run the copy step from Phase 3b manually: `cp <skill_dir>/vendor/sql-wasm.wasm <report_dir>/vendor/sql-wasm.wasm`.

**Script blocked by browser (SRI failure)**
If the browser console shows `Failed to find a valid digest in the 'integrity' attribute`, the CDN served a file that does not match the pinned hash. Do not bypass this — it means the CDN content has changed. Update the `src` URL to the correct pinned version and regenerate the `integrity` hash with `cat <file>.js | openssl dgst -sha384 -binary | base64`.
