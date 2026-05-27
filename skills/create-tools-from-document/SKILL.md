---
name: create-tools-from-document
description: >-
  Import a manufacturing tool inventory from an external document (CSV,
  spreadsheet, JSON, etc.) into a Threaded organization using the CLI. Use when
  a user has an existing tool list in any tabular format and wants to add those
  tools to Threaded.
---

# Create Tools from a Document

## Collect inputs before starting

Ask the user for these if not already provided:

- **`ORG_UUID`** (required) — Threaded organization UUID
- **`DOCUMENT_PATH`** (required) — absolute or relative path to the source file
- **`WORK_DIR`** (optional) — directory to write intermediate files; defaults to a folder named after the document stem alongside the source file

---

## Phase 1: Understand the source document

### 1a. Inspect the document

Read the source file and identify:

- **Format** — CSV, TSV, JSON array, XLSX, or other
- **Column/field names** — the exact names as they appear in the file
- **Record count** — total rows (excluding header)
- **Sample data** — read the first 5–10 rows to understand content

For CSV/TSV, use Python's `csv` module. For JSON, load and inspect the array. For XLSX, use `openpyxl` (install with `pip3 install openpyxl` if missing).

### 1b. Map document fields to CLI fields

The Threaded `tool:create` / `tool:bulk-create` command accepts these fields:

| CLI field | Type | Notes |
|---|---|---|
| `name` | string | **Required.** Tool identifier — must be unique per org. |
| `description` | string | Optional. Free-text description. |
| `cost` | string | Optional. Decimal string, e.g. `"25.00"`. Omit if not present. |
| `notes` | string | Optional. Internal notes or extra context. |

**Auto-detect required fields (apply silently, no questions needed):**

Use case-insensitive pattern matching against column names:

| Pattern matches | Maps to |
|---|---|
| `^name$`, `tool.?name`, `tool.?no`, `tool.?num` | `name` |
| `^description$`, `^desc$` | `description` |
| `^cost$`, `unit.?cost`, `^price$`, `cost.\$` | `cost` (strip `$`, commas, currency symbols) |

**For all remaining columns**, classify by data presence in the sample rows:
- Has non-empty values → propose append to `notes` (with a prefix label)
- Looks like timestamp or audit metadata (e.g. `created_at`, `updated_at`) → propose `DROP`

Common remaining column patterns:
- "Used In Operations" / "Operations" / "Process" → propose `notes` as `"Used in operations: <value>"`
- "Version" / "Rev" / "Revision" → propose `DROP` (not a tool model field)
- "Category" / "Type" → propose `notes` or `DROP`
- "Serial" / "Asset Tag" / "ID" → propose `notes` if the user wants to preserve it

### 1c. Confirm the full mapping in a single step

Present **one** mapping table covering every column — auto-detected and proposed — and wait for a single response before writing any code:

```
Proposed mapping for <filename>:

  COLUMN               FIELD          NOTE
  Name              →  name           [auto-detected, required]
  Description       →  description    [auto-detected]
  Cost ($)          →  cost           [auto-detected, strip "$"]
  Used In Ops       →  DROP           [proposed — say "notes" to keep as "Used in operations: <value>"]
  Version           →  DROP           [proposed]
  Created At        →  DROP           [proposed, timestamp metadata]

Reply with any changes, or "ok" to proceed.
```

Wait for the user's reply before proceeding. Do not ask about columns individually.

---

## Phase 2: Write the conversion script

### 2a. Create the work directory

```bash
mkdir -p <WORK_DIR>
```

### 2b. Write the script

Write a Python conversion script to `WORK_DIR/convert_to_tools_json.py`. The script must:

1. Read the source file using the appropriate library
2. Apply the confirmed field mapping
3. Omit optional fields when the source value is blank/null/empty
4. Write the output to `WORK_DIR/tools.json` as a JSON array

**Template for CSV sources:**

```python
import csv, json, sys
from pathlib import Path

SOURCE_PATH = Path("<DOCUMENT_PATH>")
OUTPUT_PATH = Path("<WORK_DIR>/tools.json")
pretty = "--pretty" in sys.argv

with open(SOURCE_PATH, newline="", encoding="utf-8-sig") as f:
    rows = list(csv.DictReader(f))

tools = []
for row in rows:
    tool = {"name": row["<name_column>"].strip()}

    # Optional fields — only include when non-empty
    desc = row.get("<description_column>", "").strip()
    if desc:
        tool["description"] = desc

    cost_raw = row.get("<cost_column>", "").strip().lstrip("$").replace(",", "")
    if cost_raw:
        tool["cost"] = cost_raw

    # Composed notes (example: append an extra column)
    extra = row.get("<extra_column>", "").strip()
    if extra:
        tool["notes"] = f"<Prefix label>: {extra}"

    tools.append(tool)

with open(OUTPUT_PATH, "w") as f:
    json.dump(tools, f, indent=2)

print(json.dumps(tools, indent=2 if pretty else None))
print(f"{len(tools)} tools generated → {OUTPUT_PATH}", file=sys.stderr)
```

Adapt the template to the actual column names and mapping. If multiple source columns contribute to `notes`, concatenate them with a separator (e.g. `"; "`).

### 2c. Run the script

```bash
python3 <WORK_DIR>/convert_to_tools_json.py --pretty 2>&1 | head -50
```

Review the first few entries. Confirm with the user that the output looks correct before proceeding.

---

## Phase 3: Dry run

Run a dry run to preview what will be created — no data is written to the database.

```bash
threaded task tool:bulk-create \
  --tools-file <WORK_DIR>/tools.json \
  --organization <ORG_UUID> \
  --dry-run
```

The dry-run output is printed to stderr. Review it with the user and confirm there are no:
- Missing required `name` fields
- Malformed `cost` values (e.g. non-numeric strings)
- Names that look like column headers (conversion bug)
- Obvious truncation or encoding issues in names/descriptions

The final stderr line is a summary:
```
Summary: N to create, N conflict(s) with existing tools
```

**If `conflicts > 0`**, choose a duplicate strategy before proceeding to the import:

| Strategy | CLI flag | When to use |
|---|---|---|
| Warn and create anyway | _(default)_ | First import; overlaps are unexpected edge cases |
| Skip existing, create new | `--skip-existing` | Re-running after a partial failure; idempotent re-runs |
| Abort and investigate | — | Unexpected overlap; audit before proceeding |

To see exactly which tool names conflict, run:
```bash
threaded task tool:list --organization <ORG_UUID> --format table
```

If issues are found, fix `convert_to_tools_json.py` and re-run the dry run until the preview looks correct.

---

## Phase 4: Execute the import

### Calling the Threaded CLI

> **Important:** `threaded` is a shell function, not a standalone binary. It must be called from a shell where the Threaded CLI has been sourced.
>
> From a terminal (normal usage):
> ```bash
> threaded task tool:bulk-create ...
> ```
>
> From a Python subprocess, use `zsh -i -c`:
> ```python
> import subprocess, json
>
> THREADED_CLI_PATH = "<path to your Threaded CLI shell aliases file>"
>
> def run_threaded(cmd_args: str) -> dict | list:
>     result = subprocess.run(
>         ["zsh", "-i", "-c", f"source {THREADED_CLI_PATH}; threaded {cmd_args}"],
>         capture_output=True, text=True,
>     )
>     output = result.stdout
>     idx_obj = output.find('{')
>     idx_arr = output.find('[')
>     candidates = [i for i in [idx_obj, idx_arr] if i != -1]
>     if not candidates:
>         raise RuntimeError(
>             f"No JSON in output (exit {result.returncode}):\n{output[:400]}\nstderr:\n{result.stderr[:400]}"
>         )
>     return json.loads(output[min(candidates):])
> ```

### Run the import

```bash
threaded task tool:bulk-create \
  --tools-file <WORK_DIR>/tools.json \
  --organization <ORG_UUID> \
  [--skip-existing] \
  --yes
```

The `--yes` flag skips the interactive confirmation prompt. Omit it if you want the CLI to show a preview and ask for confirmation before creating.

The CLI writes progress to stderr (`Creating tool N/M: "name"...`). The final JSON result contains `created`, `skipped`, and `tools` (the list of created tool records).

---

## Phase 5: Verify

### 5a. List tools

```bash
threaded task tool:list --organization <ORG_UUID> --format table
```

Confirm:
- Row count matches `created` from the import result
- Spot-check a few tool names, descriptions, and costs against the source document

### 5b. Spot-check individual records

```bash
threaded task tool:get --tool <tool_uuid>
```

Pick 3–5 tools at random, including:
- One with all optional fields populated
- One with only `name` set
- One whose source row had special characters in the name

Cross-reference against the source document to confirm the data mapped correctly.

### 5c. Report to the user

```
Import Complete
===============
Source: <filename> (<N> rows)
Organization: <ORG_UUID>
Tools created: N
Tools skipped: N
Intermediate files: <WORK_DIR>/
```

---

## Troubleshooting

**Undoing an import (no bulk-delete exists)**
There is no `tool:bulk-delete` command. To reverse an import:
1. Note the `created_at` timestamp from the import result JSON
2. Run `threaded task tool:list --organization <ORG_UUID> --format table` to get all tool UUIDs
3. Identify imported tools by their `created_at` timestamp
4. Delete individual tools through the Threaded app UI (Tools → overflow menu → Delete)

Prevention: always use `--dry-run` on production orgs and confirm `conflicts: 0` before committing.

**"No JSON in output" from CLI subprocess**
The `threaded` function wasn't sourced. Ensure the CLI source runs in the same shell invocation as the `threaded` command.

**`cost` field rejected**
The `cost` field must be a decimal string with no currency symbols or commas. Strip `$`, `£`, `,` before writing to `tools.json`.

**Duplicate name warning in output**
By default, `tool:bulk-create` warns when a tool name already exists in the org but still creates it. Use `--skip-existing` to filter those out instead.

**Encoding issues in names**
Use `encoding="utf-8-sig"` when opening CSVs exported from Excel — this strips the invisible BOM character that appears at the start of the first column name.

**Large imports (100+ tools)**
For very large imports, consider splitting the JSON into batches of ~50 and running multiple `tool:bulk-create` calls. The `--skip-existing` flag makes this safe to re-run.
