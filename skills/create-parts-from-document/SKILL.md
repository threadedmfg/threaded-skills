---
name: create-parts-from-document
description: >-
  Import a manufacturing part library from an external document (CSV,
  spreadsheet, JSON, etc.) into a Threaded organization using the CLI. Use when
  a user has an existing part list in any tabular format and wants to add those
  parts to Threaded.
---

# Create Parts from a Document

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

The Threaded `part:create` / `part:bulk-create` command accepts these fields:

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
- Has non-empty values → propose append to `notes` (with a prefix label, e.g. `"Vendor: <value>"`)
- Looks like timestamp or audit metadata (e.g. `created_at`, `updated_at`) → propose `DROP`

Common remaining column patterns:
- "Vendor" / "Supplier" / "Manufacturer" → propose `notes` as `"Vendor: <value>"`
- "Unit Cost" / "Price" / "Cost ($)" → propose `notes` as `"Cost: <value>"` (parts have no cost field)
- "Category" / "Type" / "Family" → propose `notes` or `DROP`
- "Rev" / "Revision" / "Version" → propose `notes` as `"Version: <value>"` or `DROP`
- "UOM" / "Unit of Measure" → propose `notes` or `DROP`

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

Present **one** mapping table covering every column — auto-detected and proposed — and wait for a single response before writing any code:

```
Proposed mapping for <filename>:

  COLUMN               FIELD          NOTE
  Part Number       →  part_number    [auto-detected, required]
  Description       →  description    [auto-detected]
  Make or Buy       →  make_or_buy    [auto-detected]
  Version           →  DROP           [proposed — say "notes" to keep as "Version: <value>"]
  Supplier          →  DROP           [proposed — say "notes" to keep as "Supplier: <value>"]
  Unit of Measure   →  DROP           [proposed]
  Created At        →  DROP           [proposed, timestamp metadata]
  Updated At        →  DROP           [proposed, timestamp metadata]

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

Write a Python conversion script to `WORK_DIR/convert_to_parts_json.py`. The script must:

1. Read the source file using the appropriate library
2. Apply the confirmed field mapping
3. Omit optional fields when the source value is blank/null/empty
4. Validate `make_or_buy` values and raise a clear error for unrecognized ones
5. Write the output to `WORK_DIR/parts.json` as a JSON array

**Template for CSV sources:**

```python
import csv, json, sys
from pathlib import Path

SOURCE_PATH = Path("<DOCUMENT_PATH>")
OUTPUT_PATH = Path("<WORK_DIR>/parts.json")
pretty = "--pretty" in sys.argv

MAKE_OR_BUY_MAP = {
    # adapt to the actual values in the source document
    "make": "make", "m": "make", "in-house": "make", "manufactured": "make",
    "buy": "buy", "b": "buy", "purchase": "buy", "purchased": "buy",
    "phantom": "phantom", "p": "phantom", "virtual": "phantom",
    "fabricate": "fabricate", "f": "fabricate", "fab": "fabricate",
}

with open(SOURCE_PATH, newline="", encoding="utf-8-sig") as f:
    rows = list(csv.DictReader(f))

parts = []
errors = []
for i, row in enumerate(rows, start=2):  # row 1 is the header
    part_number = row["<part_number_column>"].strip()
    if not part_number:
        errors.append(f"Row {i}: missing part_number — skipping")
        continue

    part = {"part_number": part_number}

    # Optional fields — only include when non-empty
    desc = row.get("<description_column>", "").strip()
    if desc:
        part["description"] = desc

    mob_raw = row.get("<make_or_buy_column>", "").strip().lower()
    if mob_raw:
        mob = MAKE_OR_BUY_MAP.get(mob_raw)
        if mob is None:
            errors.append(f"Row {i}: unrecognized make_or_buy value '{mob_raw}'")
        else:
            part["make_or_buy"] = mob

    # Composed notes (example: append an extra column)
    extra = row.get("<extra_column>", "").strip()
    if extra:
        part["notes"] = f"<Prefix label>: {extra}"

    parts.append(part)

if errors:
    print("\n".join(errors), file=sys.stderr)

with open(OUTPUT_PATH, "w") as f:
    json.dump(parts, f, indent=2)

print(json.dumps(parts, indent=2 if pretty else None))
print(f"{len(parts)} parts generated → {OUTPUT_PATH}", file=sys.stderr)
```

Adapt the template to the actual column names and mapping. If multiple source columns contribute to `notes`, concatenate them with a separator (e.g. `"; "`).

### 2c. Run the script

```bash
python3 <WORK_DIR>/convert_to_parts_json.py --pretty 2>&1 | head -60
```

Review the first few entries. Look for:
- Any `make_or_buy` errors reported to stderr
- Rows skipped due to missing `part_number`
- Unexpected `null` values or empty strings

Confirm with the user that the output looks correct before proceeding.

---

## Phase 3: Dry run

Run a dry run to preview what will be created — no data is written to the database.

```bash
threaded task part:bulk-create \
  --parts-file <WORK_DIR>/parts.json \
  --organization <ORG_UUID> \
  --dry-run
```

The dry-run output is printed to stderr. Review it with the user and confirm there are no:
- Missing required `part_number` fields
- Invalid `make_or_buy` values (CLI will exit with an error)
- Part numbers that look like column headers (conversion bug)
- Obvious truncation or encoding issues in part numbers or descriptions

The final stderr line is a summary:
```
Summary: N to create, N conflict(s) with existing parts
```

**If `conflicts > 0`**, choose a duplicate strategy before proceeding to the import:

| Strategy | CLI flag | When to use |
|---|---|---|
| Warn and create anyway | _(default)_ | First import; overlaps are unexpected edge cases |
| Skip existing, create new | `--skip-existing` | Re-running after a partial failure; idempotent re-runs |
| Abort and investigate | — | Unexpected overlap; audit before proceeding |

To see exactly which part numbers conflict, run:
```bash
threaded task part:list --organization <ORG_UUID> --format table
```

If issues are found, fix `convert_to_parts_json.py` and re-run the dry run until the preview looks correct.

---

## Phase 4: Execute the import

### Calling the Threaded CLI

> **Important:** `threaded` is a shell function, not a standalone binary. It must be called from a shell where the Threaded CLI has been sourced.
>
> From a terminal (normal usage):
> ```bash
> threaded task part:bulk-create ...
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
threaded task part:bulk-create \
  --parts-file <WORK_DIR>/parts.json \
  --organization <ORG_UUID> \
  [--skip-existing] \
  --yes
```

The `--yes` flag skips the interactive confirmation prompt. Omit it if you want the CLI to show a preview and ask for confirmation before creating.

The CLI writes progress to stderr (`Creating part N/M: "part_number"...`). The final JSON result contains `created`, `skipped`, and `parts` (the list of created part records).

---

## Phase 5: Verify

### 5a. List parts

```bash
threaded task part:list --organization <ORG_UUID> --format table
```

Confirm:
- Row count matches `created` from the import result
- Spot-check a few part numbers, descriptions, and `make_or_buy` values against the source document

Filter by part number to quickly find a specific record:

```bash
threaded task part:list --organization <ORG_UUID> --part-number <search_text>
```

### 5b. Spot-check individual records

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
Intermediate files: <WORK_DIR>/
```

---

## Troubleshooting

**Undoing an import (no bulk-delete exists)**
There is no `part:bulk-delete` command. To reverse an import:
1. Note the `created_at` timestamp from the import result JSON
2. Run `threaded task part:list --organization <ORG_UUID> --format table` to get all part UUIDs
3. Identify imported parts by their `created_at` timestamp
4. Delete individual parts through the Threaded app UI (Parts → overflow menu → Delete)

Prevention: always use `--dry-run` on production orgs and confirm `conflicts: 0` before committing.

**`make_or_buy` value rejected by CLI**
The CLI validates `make_or_buy` before sending to the API. Accepted values are `make`, `buy`, `phantom`, `fabricate` (lowercase). Check `convert_to_parts_json.py` — the mapping dict may be missing an entry. Re-run the script after fixing.

**"No JSON in output" from CLI subprocess**
The `threaded` function wasn't sourced. Ensure the CLI source runs in the same shell invocation as the `threaded` command.

**Duplicate part_number warning in output**
By default, `part:bulk-create` warns when a part number already exists in the org but still creates it. Use `--skip-existing` to filter those out instead.

**Rows with blank `part_number`**
The conversion script should skip these rows and report them. Part number is required — blank entries cannot be imported.

**Encoding issues in part numbers**
Use `encoding="utf-8-sig"` when opening CSVs exported from Excel — this strips the invisible BOM character that appears at the start of the first column name.

**`make_or_buy` column is missing entirely**
Omit the field from the output JSON entirely. The API defaults to `make`. Confirm with the user that this default is correct for their dataset before proceeding.

**Large imports (100+ parts)**
For very large imports, consider splitting the JSON into batches of ~50 and running multiple `part:bulk-create` calls. The `--skip-existing` flag makes this safe to re-run.
