---
name: update-threaded-report
description: >-
  Full update pipeline for any Threaded report group: rebuild the SQLite database
  from source data, generate AI insights, push shell and data revisions. Works
  with any customer repo that has a report-config.yaml. Use when refreshing
  reports after source data changes.
---

# Update Threaded Report

> **Before starting:** Read the `REQUIRED-threaded-runner-setup` skill to determine whether you are running via MCP or CLI, check auth status, and confirm the organization UUID.

## Collect inputs before starting

Ask the user for these if not already provided:

- **`--config`** (optional) — path to `report-config.yaml`. If omitted, auto-discover by searching the current directory and parent directories.
- **`--data-only`** (optional) — rebuild DB and push data only; skip insights generation and shell push.
- **`--insights-only`** (optional) — regenerate insights and push shells only; skip DB rebuild and data push.

---

## Step 1: Verify CLI authentication

Check that the Threaded CLI is authenticated and targeting the correct organization.

**CLI:**
```bash
threaded auth status
```

If `threaded` is only available as a shell alias/function:
```bash
zsh -ic 'threaded auth status 2>&1'
```

Check for the `THREADED_CLI_COMMAND` environment variable first — if set, use it for all CLI invocations. Otherwise fall back to `zsh -ic 'threaded ...'`.

Confirm:
- `loggedIn: true`
- The active organization UUID matches `organization_uuid` from `report-config.yaml`

If not authenticated, ask the user to log in before continuing.

### Load the config

Parse `report-config.yaml`. Extract all report entries and the `db` section. Resolve all paths relative to the config file's directory.

---

## Step 2: Rebuild the database

**Skip this step if `--insights-only` is set.**

### 2a. Reset and create schema

Run the database creation script with `--reset` to drop and recreate all tables:

```bash
python3 <db.create_script> --reset
```

### 2b. Load data from source

The config's `db` section specifies how to load data. Two patterns are supported:

**Module-based loader** (when `loader_module` is set):
```bash
PYTHONPATH=<db.loader_pythonpath> python3 -m <db.loader_module>
```

**Script-based loader** (when `loader_script` is set):
```bash
python3 <db.loader_script>
```

### 2c. Validate the database

Run row count checks on key tables and a foreign key integrity check:

```bash
python3 -c "
import sqlite3
conn = sqlite3.connect('<db_path>')

# Row counts
tables = conn.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()
for t in tables:
    count = conn.execute(f'SELECT COUNT(*) FROM [{t[0]}]').fetchone()[0]
    print(f'{t[0]}: {count} rows')
    if count == 0:
        print(f'  ⚠ WARNING: {t[0]} is empty')

# Foreign key check
fk_errors = conn.execute('PRAGMA foreign_key_check').fetchall()
if fk_errors:
    print(f'FK errors: {fk_errors}')
    raise SystemExit('Foreign key check failed')
else:
    print('Foreign key check: OK')

conn.close()
"
```

**Stop and investigate** if any core table has zero rows or the foreign key check fails. Do not proceed to push data with a broken database.

---

## Step 3: Generate insights

**Skip this step if `--data-only` is set.**

Use the `generate-report-insights` skill to generate fresh insights for all reports in the config. If that skill is installed:

```
generate-report-insights --config <config_path>
```

If the skill is not installed, follow its workflow inline:

1. For each report in the config:
   - Read the `.insights-prompt.md` template
   - Run the companion `.sql` queries against the database
   - Run the `summarize_script` if available
   - Substitute `{LOAD_DATE}`, `{SUMMARY_OUTPUT}`, `{QUERY_RESULTS}` into the template
   - Append output format instructions
   - Call the LLM and validate the response
   - Write the `.insights.js` file

---

## Step 4: Push shell revisions

**Skip this step if `--data-only` is set.**

### 4a. Inject insights into HTML

For each report, run `build_insights.py` to inject the `.insights.js` content between `// ==INSIGHTS-BEGIN==` and `// ==INSIGHTS-END==` markers in the HTML shell.

Look for `build_insights.py` in a `scripts/` directory near the config file:

```bash
python3 <scripts_dir>/build_insights.py --html <shell_path> --insights <insights_js_path>
```

### 4b. Push shells to Threaded

For each report, push the updated HTML shell:

```bash
threaded task org:report:shell-revision \
  --organization <org_uuid> \
  --report <report_uuid> \
  --shell-path <shell_path>
```

Or, if a `push_shell.sh` script exists in the report group's `scripts/` directory, prefer calling it:

```bash
bash <scripts_dir>/push_shell.sh
```

The push scripts handle CLI invocation internally (checking `THREADED_CLI_COMMAND`, monorepo paths, etc.).

---

## Step 5: Push data revisions

**Skip this step if `--insights-only` is set.**

For each report in the config, push the rebuilt database as a new data revision:

```bash
threaded task org:report:upload-new-data \
  --organization <org_uuid> \
  --report <report_uuid> \
  --data-path <db_path>
```

Or, if a `push_data.sh` script exists:

```bash
bash <scripts_dir>/push_data.sh
```

Note: multiple reports may share the same database file. The push scripts typically handle uploading to all reports in one call.

---

## Step 6: Verify

### 6a. Confirm data revision uploaded

List reports and check that each target report has a latest data revision with the correct `data_file`:

```bash
threaded task org:reports:list --organization <org_uuid>
```

### 6b. Download and spot-check

Download the latest data revision for one report and query it locally:

```bash
threaded task org:report:download \
  --organization <org_uuid> \
  --report <first_report_uuid> \
  --output /tmp/verify_report_data.db
```

```bash
python3 -c "
import sqlite3
conn = sqlite3.connect('/tmp/verify_report_data.db')
tables = conn.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()
for t in tables[:3]:
    count = conn.execute(f'SELECT COUNT(*) FROM [{t[0]}]').fetchone()[0]
    print(f'{t[0]}: {count} rows')
conn.close()
"
```

Confirm row counts match what was loaded in Step 2.

### 6c. Report results

```
Update Complete
===============
Config: <config_path>
Organization: <org_uuid>
Reports updated: <count>
  - <report_name>: shell rev <N>, data rev <N>
  - ...
Database: <row_count> total rows across <table_count> tables
Insights: <keys_generated> keys generated per report
```

---

## Usage examples

```bash
# Full update: rebuild DB + generate insights + push everything
update-threaded-report

# Rebuild DB and push data only (skip insights)
update-threaded-report --data-only

# Regenerate insights and push shells only (skip DB rebuild)
update-threaded-report --insights-only

# Explicit config path
update-threaded-report --config /path/to/report-config.yaml
```

---

## Troubleshooting

**CLI not authenticated**
Run `threaded auth login` or `zsh -ic 'threaded auth login'` to authenticate. Confirm the correct organization UUID is active.

**Database rebuild fails**
Check that the source data files exist (workbook, CSVs, etc.) and that Python dependencies are installed (`openpyxl` for XLSX, `csv` for CSV). Run `pip3 install openpyxl` if needed.

**Push fails with "report not found"**
Verify the report UUIDs in `report-config.yaml` match the live reports. Run `threaded task org:reports:list` to check.

**Shell push fails with "shell file not found"**
The HTML shell path in the config must point to an existing file. Check that `build_insights.py` ran successfully and didn't corrupt the HTML.

**Foreign key check fails after reload**
This usually means the source data has referential integrity issues (e.g., a BOM references a part number that doesn't exist in the parts table). Check the loader output for warnings.

**Insights generation fails**
See the `generate-report-insights` skill troubleshooting section. Common causes: missing `.insights-prompt.md` file, missing companion `.sql` file, or database not fully built.
