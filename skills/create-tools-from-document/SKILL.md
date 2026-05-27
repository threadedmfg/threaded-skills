---
name: create-tools-from-document
description: >-
  Import a manufacturing tool inventory from an external document (CSV,
  spreadsheet, JSON, etc.) into a Threaded organization. Works with the
  Threaded MCP server or directly with the Threaded CLI. Use when a user has
  an existing tool list in any tabular format and wants to add those tools to
  Threaded.
---

# Create Tools from a Document

> **Before starting:** Read `skills/SHARED.md` (or the `threaded-runner-setup` skill) to determine whether you are running via MCP or CLI, check auth status, and confirm the organization UUID.

## Collect inputs before starting

Ask the user for these if not already provided:

- **`ORG_UUID`** (required) — Threaded organization UUID
- **`DOCUMENT_PATH`** (required) — absolute or relative path to the source file

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

The Threaded `tool:bulk-create` command accepts these fields:

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

Present **one** mapping table covering every column — auto-detected and proposed — and wait for a single response before proceeding:

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

## Phase 2: Generate the tools JSON

Parse the source file and produce a JSON array of tool objects matching the confirmed mapping. No intermediate files are needed — generate the array directly.

Use the appropriate library to read the source:
- **CSV/TSV:** Python's `csv.DictReader` (use `encoding="utf-8-sig"` for Excel exports)
- **JSON:** load directly
- **XLSX:** `openpyxl` (`pip3 install openpyxl` if missing)

Apply the confirmed mapping:
- Omit optional fields when the source value is blank/null/empty
- Strip currency symbols and commas from `cost` values — must be a plain decimal string (e.g. `"25.00"`)
- Concatenate multiple source columns into `notes` with a separator (e.g. `"; "`)

Preview the first 5 entries and the total count. Confirm with the user that the output looks correct before running any Threaded commands.

**Expected output shape:**
```json
[
  {"name": "Torque Wrench 3/8\"", "description": "3/8 inch drive", "cost": "45.00"},
  {"name": "Hex Key Set", "notes": "Used in operations: Final Assembly"},
  ...
]
```

Store this array as `TOOLS_JSON` for use in the following phases.

---

## Phase 3: Dry run

Run a dry run to preview what will be created — no data is written to the database.

**MCP:**
```
execute_threaded_script(script="threaded task tool:bulk-create --tools '<TOOLS_JSON>' --organization <ORG_UUID> --dry-run")
```

**CLI:**
```bash
threaded task tool:bulk-create \
  --tools '<TOOLS_JSON>' \
  --organization <ORG_UUID> \
  --dry-run
```

The dry-run output is printed to stderr. Review it with the user and confirm there are no:
- Missing required `name` fields
- Malformed `cost` values (e.g. non-numeric strings)
- Names that look like column headers (mapping bug)
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

To see exactly which tool names conflict:

**MCP:**
```
execute_threaded_script(script="threaded task tool:list --organization <ORG_UUID> --format table")
```

**CLI:**
```bash
threaded task tool:list --organization <ORG_UUID> --format table
```

If issues are found, fix the mapping in Phase 2 and re-run the dry run until the preview looks correct.

---

## Phase 4: Execute the import

**MCP:**
```
execute_threaded_script(script="threaded task tool:bulk-create --tools '<TOOLS_JSON>' --organization <ORG_UUID> [--skip-existing] --yes")
```

**CLI:**
```bash
threaded task tool:bulk-create \
  --tools '<TOOLS_JSON>' \
  --organization <ORG_UUID> \
  [--skip-existing] \
  --yes
```

The `--yes` flag skips the interactive confirmation prompt.

The final JSON result contains `created`, `skipped`, and `tools` (the list of created tool records).

---

## Phase 5: Verify

### 5a. List tools

**MCP:**
```
execute_threaded_script(script="threaded task tool:list --organization <ORG_UUID> --format table")
```

**CLI:**
```bash
threaded task tool:list --organization <ORG_UUID> --format table
```

Confirm:
- Row count matches `created` from the import result
- Spot-check a few tool names, descriptions, and costs against the source document

### 5b. Spot-check individual records

**MCP:**
```
execute_threaded_script(script="threaded task tool:get --tool <tool_uuid>")
```

**CLI:**
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
```

---

## Troubleshooting

**Undoing an import (no bulk-delete exists)**
There is no `tool:bulk-delete` command. To reverse an import:
1. Note the `created_at` timestamp from the import result JSON
2. Run `tool:list` to get all tool UUIDs
3. Identify imported tools by their `created_at` timestamp
4. Delete individual tools through the Threaded app UI (Tools → overflow menu → Delete)

Prevention: always use `--dry-run` on production orgs and confirm `conflicts: 0` before committing.

**`cost` field rejected**
The `cost` field must be a decimal string with no currency symbols or commas. Strip `$`, `£`, `,` before including in the JSON.

**Duplicate name warning in output**
By default, `tool:bulk-create` warns when a tool name already exists in the org but still creates it. Use `--skip-existing` to filter those out instead.

**Encoding issues in names**
Use `encoding="utf-8-sig"` when opening CSVs exported from Excel — this strips the invisible BOM character that appears at the start of the first column name.

**Large imports (100+ tools)**
Split the JSON into batches of ~50 and run multiple `tool:bulk-create` calls. The `--skip-existing` flag makes this safe to re-run across batches.
