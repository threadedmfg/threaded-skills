---
name: create-parts-from-document
description: >-
  Import a manufacturing part library from an external document (CSV,
  spreadsheet, JSON, etc.) into a Threaded organization. Works with the
  Threaded MCP server or directly with the Threaded CLI. Use when a user has
  an existing part list in any tabular format and wants to add those parts to
  Threaded.
---

# Create Parts from a Document

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

The Threaded `part:bulk-create` command accepts these fields:

| CLI field | Type | Notes |
|---|---|---|
| `part_number` | string | **Required.** Part identifier — must be unique per org. |
| `description` | string | Optional. Free-text description. |
| `make_or_buy` | string | Optional. One of: `make`, `buy`, `phantom`, `fabricate`. Defaults to `make` if omitted. |
| `notes` | string | Optional. Internal notes or extra context. |

**Auto-detect required fields (apply silently, no questions needed):**

Use case-insensitive pattern matching against column names:

| Pattern matches | Maps to |
|---|---|
| `part.?num`, `part.?no`, `^pn$`, `item.?no` | `part_number` |
| `^description$`, `^desc$`, `item.?desc` | `description` |
| `make.?or.?buy`, `make.?buy`, `^mob$`, `^source$`, `^procurement$` | `make_or_buy` |

**For all remaining columns**, classify by data presence in the sample rows:
- Has non-empty values → propose append to `notes` (with a prefix label, e.g. `"Vendor: <value>"`). Customer-visible business fields such as vendor, cost, and category should default to `notes` unless the user chooses to drop them.
- Looks like timestamp or audit metadata (e.g. `created_at`, `updated_at`) → propose `DROP`

Common remaining column patterns:
- "Vendor" / "Supplier" / "Manufacturer" → propose `notes` as `"Vendor: <value>"`
- "Unit Cost" / "Price" / "Cost ($)" → propose `notes` as `"Cost: <value>"` (parts have no cost field)
- "Category" / "Type" / "Family" → propose `notes` as `"Category: <value>"`
- "Rev" / "Revision" / "Version" → propose `notes` as `"Version: <value>"`
- "UOM" / "Unit of Measure" → propose `notes` as `"UOM: <value>"`

**Mapping `make_or_buy` values:**

The `make_or_buy` field is an enum. Translate source values to one of the four valid Threaded values:

| Common source value | Threaded value |
|---|---|
| "Make" / "M" / "In-house" / "Manufactured" / "Internal" | `make` |
| "Buy" / "B" / "Purchase" / "Purchased" / "Procure" / "External" | `buy` |
| "Phantom" / "P" / "Virtual" / "Assembly" | `phantom` |
| "Fabricate" / "F" / "Fab" / "Custom" / "Custom Fab" | `fabricate` |
| (blank / missing) | omit field — Threaded defaults to `make` |

If a source value doesn't match any pattern above, flag it to the user rather than guessing.

### 1c. Confirm the full mapping in a single step

Present **one** mapping table covering every column — auto-detected and proposed — and wait for a single response before proceeding:

```
Proposed mapping for <filename>:

  COLUMN               FIELD          NOTE
  Part Number       →  part_number    [auto-detected, required]
  Description       →  description    [auto-detected]
  Make or Buy       →  make_or_buy    [auto-detected]
  Supplier          →  notes          [proposed as "Vendor: <value>" — say "drop" to exclude]
  Version           →  notes          [proposed as "Version: <value>" — say "drop" to exclude]
  Unit of Measure   →  notes          [proposed as "UOM: <value>" — say "drop" to exclude]
  Created At        →  DROP           [proposed, timestamp metadata]
  Updated At        →  DROP           [proposed, timestamp metadata]

Reply with any changes, or "ok" to proceed.
```

Wait for the user's reply before proceeding. Do not ask about columns individually.

---

## Phase 2: Generate the parts JSON

Parse the source file and produce a JSON array of part objects matching the confirmed mapping. No intermediate files are needed — generate the array directly.

Use the appropriate library to read the source:
- **CSV/TSV:** Python's `csv.DictReader` (use `encoding="utf-8-sig"` for Excel exports)
- **JSON:** load directly
- **XLSX:** `openpyxl` (`pip3 install openpyxl` if missing)

Apply the confirmed mapping:
- Omit optional fields when the source value is blank/null/empty
- Skip rows where `part_number` is blank and report them to the user
- Translate `make_or_buy` source values to the Threaded enum values; flag any unrecognized values before proceeding
- Concatenate multiple source columns into `notes` with a separator (e.g. `"; "`)

Preview the first 5 entries, the total count, and any skipped/flagged rows. Confirm with the user that the output looks correct before running any Threaded commands.

**Expected output shape:**
```json
[
  {"part_number": "PN-001", "description": "Lid assembly", "make_or_buy": "make"},
  {"part_number": "PN-002", "make_or_buy": "buy", "notes": "Vendor: Acme Corp"},
  ...
]
```

Store this array as `PARTS_JSON` for use in the following phases.

---

## Phase 3: Dry run

**Always run a dry run before importing.** No data is written to the database during this phase.

**MCP:**
```
execute_threaded_script(script="threaded task part:bulk-create --parts '<PARTS_JSON>' --organization <ORG_UUID> --dry-run")
```

**CLI:**
```bash
threaded task part:bulk-create \
  --parts '<PARTS_JSON>' \
  --organization <ORG_UUID> \
  --dry-run
```

The dry-run output is printed to stderr. Review it with the user and confirm there are no:
- Missing required `part_number` fields
- Invalid `make_or_buy` values (CLI will exit with an error)
- Part numbers that look like column headers (mapping bug)
- Obvious truncation or encoding issues in part numbers or descriptions

The final stderr line is a summary:
```
Summary: N to create, N conflict(s) with existing parts
```

**If `conflicts > 0`**, stop and ask the user to choose a duplicate strategy before proceeding. Do not run the import until the user has explicitly confirmed their choice:

| Strategy | CLI flag | When to use |
|---|---|---|
| Skip existing, create new only | `--skip-existing` | Re-running after a partial failure; recommended safe default |
| Warn and create anyway (may create duplicates) | _(default, no flag)_ | Only if the user explicitly accepts that duplicates may be created |
| Abort and investigate | — | Unexpected overlap; list existing parts to audit before proceeding |

To see exactly which part numbers conflict:

**MCP:**
```
execute_threaded_script(script="threaded task part:list --organization <ORG_UUID> --format table")
```

**CLI:**
```bash
threaded task part:list --organization <ORG_UUID> --format table
```

If issues are found, fix the mapping in Phase 2 and re-run the dry run until the preview looks correct. **Do not proceed to Phase 4 until the user has approved the dry-run preview.**

---

## Phase 4: Execute the import

Only run the import after the user has confirmed the dry-run preview and, if there were conflicts, chosen a duplicate strategy.

**MCP:**
```
execute_threaded_script(script="threaded task part:bulk-create --parts '<PARTS_JSON>' --organization <ORG_UUID> [--skip-existing] --yes")
```

**CLI:**
```bash
threaded task part:bulk-create \
  --parts '<PARTS_JSON>' \
  --organization <ORG_UUID> \
  [--skip-existing] \
  --yes
```

The `--yes` flag skips the interactive confirmation prompt. Only add it after the user has confirmed the import.

The final JSON result contains `created`, `skipped`, and `parts` (the list of created part records).

---

## Phase 5: Verify

### 5a. List parts

**MCP:**
```
execute_threaded_script(script="threaded task part:list --organization <ORG_UUID> --format table")
```

**CLI:**
```bash
threaded task part:list --organization <ORG_UUID> --format table
```

Confirm:
- Row count matches `created` from the import result
- Spot-check a few part numbers, descriptions, and `make_or_buy` values against the source document

Filter by part number to quickly find a specific record:

**MCP:**
```
execute_threaded_script(script="threaded task part:list --organization <ORG_UUID> --part-number <search_text>")
```

**CLI:**
```bash
threaded task part:list --organization <ORG_UUID> --part-number <search_text>
```

### 5b. Spot-check individual records

**MCP:**
```
execute_threaded_script(script="threaded task part:get --part <part_uuid>")
```

**CLI:**
```bash
threaded task part:get --part <part_uuid>
```

Pick 3–5 parts at random, including:
- One with all optional fields populated
- One with only `part_number` set
- One whose source row had a non-`make` `make_or_buy` value
- One whose source row had special characters in the part number

Cross-reference against the source document to confirm the data mapped correctly.

### 5c. Report to the user

```
Import Complete
===============
Source: <filename> (<N> rows)
Organization: <ORG_UUID>
Parts created: N
Parts skipped: N
```

---

## Troubleshooting

**Undoing an import (no bulk-delete exists)**
There is no `part:bulk-delete` command. To reverse an import:
1. Note the `created_at` timestamp from the import result JSON
2. Run `part:list` to get all part UUIDs
3. Identify imported parts by their `created_at` timestamp
4. Delete individual parts through the Threaded app UI (Parts → overflow menu → Delete)

Prevention: always use `--dry-run` on production orgs and confirm `conflicts: 0` before committing.

**`make_or_buy` value rejected by CLI**
The CLI validates `make_or_buy` before sending to the API. Accepted values are `make`, `buy`, `phantom`, `fabricate` (lowercase). Check that all source values have been translated correctly in Phase 2.

**Duplicate part_number warning in output**
By default, `part:bulk-create` warns when a part number already exists in the org but still creates it. Use `--skip-existing` to filter those out instead.

**Rows with blank `part_number`**
Skip these rows and report them. Part number is required — blank entries cannot be imported.

**Encoding issues in part numbers**
Use `encoding="utf-8-sig"` when opening CSVs exported from Excel — this strips the invisible BOM character that appears at the start of the first column name.

**`make_or_buy` column is missing entirely**
Omit the field from the output JSON entirely. The API defaults to `make`. Confirm with the user that this default is correct for their dataset before proceeding.

**Large imports (100+ parts)**
Split the JSON into batches of ~50 and run multiple `part:bulk-create` calls. The `--skip-existing` flag makes this safe to re-run across batches.
