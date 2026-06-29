---
name: generate-report-insights
description: >-
  Generate AI-powered insights for Threaded reports using report-config.yaml,
  companion SQL queries, and per-report prompt templates. Reads the SQLite
  database, runs queries, builds an LLM prompt, and writes validated
  .insights.js files. Use when refreshing report insights after a data update.
---

# Generate Report Insights

> **Before starting:** Read the `REQUIRED-threaded-runner-setup` skill to determine whether you are running via MCP or CLI, check auth status, and confirm the organization UUID.

## Collect inputs before starting

Ask the user for these if not already provided:

- **`--config`** (optional) — path to `report-config.yaml`. If omitted, auto-discover by searching the current directory and parent directories for a file named `report-config.yaml`.
- **`--report`** (optional) — name of a single report to target (must match a `name` in the config). If omitted, generate insights for all reports in the config.
- **`--push`** (optional) — if set, inject insights into HTML shells and push shell revisions to Threaded after generation.

---

## Phase 1: Read config and verify prerequisites

### 1a. Load the config

Parse `report-config.yaml` with a YAML parser. Extract:

- `organization_uuid`
- The `reports` list (each entry has `name`, `slug`, `uuid`, `shell`, `insights_js`, `insights_prompt`, `queries_sql`, `db`, `insight_keys`)
- The `db` section (optional fields: `summarize_script`)

If `--report` was specified, filter the reports list to only the matching entry. Error if no match is found.

All paths in the config are **relative to the config file's directory**. Resolve them to absolute paths before use.

### 1b. Verify the database exists and is non-empty

For each target report, check that the `db` file exists. Then run a quick sanity check:

```bash
python3 -c "
import sqlite3, sys
conn = sqlite3.connect('<db_path>')
tables = conn.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()
print(f'Tables: {len(tables)}')
for t in tables[:5]:
    count = conn.execute(f'SELECT COUNT(*) FROM [{t[0]}]').fetchone()[0]
    print(f'  {t[0]}: {count} rows')
conn.close()
"
```

Stop if the database file is missing or if key tables have zero rows.

---

## Phase 2: Gather data for each target report

For each report in the target list:

### 2a. Read the insights prompt template

Read the `.insights-prompt.md` file specified by the report's `insights_prompt` path. This file contains business context, per-key analysis questions, format requirements, and data caveats.

### 2b. Run companion SQL queries

Read the `.sql` file specified by the report's `queries_sql` path. SQL files contain multiple queries separated by comment headers using this pattern:

```sql
-- ── N. Query title ───────────────────────────────────────────────────────
SELECT ...
```

Split the file on `-- ──` markers. For each query:

1. Extract the title from the comment line
2. Execute the query against the database
3. Capture the results as a list of dictionaries (column names as keys)

Store all results as a JSON object keyed by query title:

```json
{
  "1. Overall summary": [{"total_builds": 1563, "on_track": 634, ...}],
  "2. Builds by month": [{"month": "2026-01", "total": 65, ...}, ...],
  ...
}
```

If `queries_sql` is null or the file doesn't exist, set `{QUERY_RESULTS}` to `"No companion queries available."`.

### 2c. Run the summarize script (if available)

If the `db` section of the config has a `summarize_script` that is non-null and the file exists:

```bash
python3 <summarize_script>
```

Capture stdout as `{SUMMARY_OUTPUT}`. If no summarize script exists, set `{SUMMARY_OUTPUT}` to `"No summary script available."`.

---

## Phase 3: Build the LLM prompt

### 3a. Substitute template variables

In the insights prompt template, replace:

| Variable | Value |
|---|---|
| `{LOAD_DATE}` | Today's date (YYYY-MM-DD) |
| `{SUMMARY_OUTPUT}` | Output from the summarize script (or placeholder) |
| `{QUERY_RESULTS}` | JSON string of all query results |

### 3b. Append output format instructions

After the template content, append:

```
Return ONLY a valid JavaScript const declaration with this exact format:

const INSIGHTS = {
  'key': { title: 'string', body: `html string` },
  ...
};

Include exactly these keys: [list from insight_keys in config]

Rules:
- Do not include any markdown, explanation, code fences, or wrapping — only the JS
- Use backtick template literals for body values
- Body content must use HTML tags only (no markdown): <h4> for headings, <p> for paragraphs, <ul><li> for lists, <strong> for emphasis, <div class="insight-note"> for caveats
- Limit each key's body to approximately 150 words
- Include specific numbers, dates, and percentages from the query results
- End each body with a <div class="insight-note"> containing data freshness or methodology caveats
```

---

## Phase 4: Generate and validate

### 4a. Call the LLM

Send the assembled prompt as a user message. The response should contain only the JavaScript `const INSIGHTS = { ... };` declaration.

### 4b. Parse the INSIGHTS object

Extract the text between `const INSIGHTS = {` and the matching closing `};` (inclusive of the declaration). Handle cases where the LLM wraps the response in code fences — strip any `` ```javascript `` or `` ``` `` markers.

### 4c. Validate

Check all of the following:

| Check | Failure action |
|---|---|
| All expected keys (from `insight_keys` in config) are present | Retry |
| No unexpected extra keys | Retry |
| Each key has non-empty `title` (string) | Retry |
| Each key has non-empty `body` (string) | Retry |
| The output is syntactically valid JavaScript | Retry |

If validation fails, retry **once** with the validation error prepended to the prompt:

```
The previous response had validation errors:
- Missing key: 'gantt'
- Extra key: 'chart'

Please fix and return the corrected output.
```

If validation fails a second time, stop and present the errors to the user for manual resolution.

---

## Phase 5: Write output

### 5a. Write the insights file

Write the validated JavaScript to the `insights_js` path from the config. This overwrites any existing file.

### 5b. Report results

For each generated report, print:

```
✓ Generated insights for: <report name>
  Keys: title, stats, gantt, table
  Word counts: title (142), stats (138), gantt (127), table (145)
  Output: <insights_js path>
```

---

## Phase 6: Optional push (if `--push`)

If the `--push` flag was set:

### 6a. Inject insights into HTML shells

For each target report, run the `build_insights.py` script to inject the `.insights.js` content into the HTML shell between `// ==INSIGHTS-BEGIN==` and `// ==INSIGHTS-END==` markers:

```bash
python3 <build_insights_script> --html <shell_path> --insights <insights_js_path>
```

If `build_insights.py` is in the same directory tree, locate it by searching the config's parent directories for `scripts/build_insights.py` or `build_insights.py`.

### 6b. Push shell revisions

For each target report, push the updated HTML shell to Threaded:

**MCP:**
```
execute_threaded_script(script="threaded task org:report:shell-revision --organization <org_uuid> --report <report_uuid> --shell-path <shell_path>")
```

**CLI:**
```bash
threaded task org:report:shell-revision \
  --organization <org_uuid> \
  --report <report_uuid> \
  --shell-path <shell_path>
```

Or, if a `push_shell.sh` script exists in the report group's `scripts/` directory, prefer calling it directly as it handles CLI invocation internally.

---

## Usage examples

```bash
# Generate insights for all reports in the current directory's config
generate-report-insights

# Target a specific report
generate-report-insights --report "Build Status"

# Generate and push shell revisions immediately
generate-report-insights --push

# Explicit config path
generate-report-insights --config /path/to/report-config.yaml
```

---

## Troubleshooting

**Database file not found**
Verify the `db` path in `report-config.yaml` is correct and relative to the config file's directory. Run the DB rebuild pipeline first if the database hasn't been created yet.

**SQL query errors**
Companion `.sql` files may reference views that only exist after the database has been fully built (schema + loaders). Rebuild the database if views are missing.

**LLM output validation failures**
Common issues: extra keys (LLM adds commentary keys), markdown in body (use HTML only), missing keys (LLM combines sections). The retry mechanism handles most cases. If persistent, check that the prompt template has clear per-key sections.

**`build_insights.py` not found**
The injection script must exist in the report group's directory tree. Check `scripts/build_insights.py` relative to the config file.

**Insights markers missing from HTML**
The HTML shell must contain `// ==INSIGHTS-BEGIN==` and `// ==INSIGHTS-END==` comment markers in its `<script>` block. If missing, add them manually around the `const INSIGHTS = { ... };` declaration.
