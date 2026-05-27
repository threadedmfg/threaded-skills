---
name: create-work-instructions-from-document
description: >-
  Create work instructions in Threaded from a document showing a manufacturing
  or assembly process. Extracts images from the document, analyzes images and text
  to identify procedures and steps, authors the work instruction structure, and imports it
  into Threaded with reference images attached as visuals. Use when a user has a
  document demonstrating a process and wants to turn it into Threaded work
  instructions. MCP users can create folder and WI structure; image upload and
  visual attachment require the Threaded CLI.
---

# Create Work Instructions from a Document

> **Before starting:** Read `skills/SHARED.md` (or the `threaded-runner-setup` skill) to determine whether you are running via MCP or CLI, check auth status, and confirm the organization UUID.

> **MCP users — media upload limitation:** Uploading images and attaching visuals (Phases 7 and 9) requires the Threaded CLI. If you are running via MCP only, you can complete Phases 1–6 and Phase 8 to create the folder and WI structure, but you will need CLI access to upload images and attach them as visuals.

> **Data privacy:** This skill reads the full text and images from the source document, writes intermediate files to `WORK_DIR`, and uploads images to Threaded. Confirm that the AI agent is approved to process this document before proceeding. Review the generated JSON in Phase 6 before importing. Clean up `WORK_DIR` after a successful import if you do not want intermediate files retained on disk.

## Collect inputs before starting

Ask the user for these if not already provided:

- **`ORG_UUID`** (required) — Threaded organization UUID
- **`DOCUMENT_PATH`** (required) — absolute or relative path to the document file (PDF, DOCX, images, etc.)
- **`WORK_DIR`** (optional) — directory to write intermediate files; defaults to a folder named after the document stem alongside the document file
- **`WI_NAME`** (optional) — name for the work instruction; if not provided, infer from the document content after inspecting it
- **`FOLDER_NAME`** (optional) — name for the WI folder in Threaded; defaults to `WI_NAME` if not provided

---

## Phase 1: Inspect the Document

1. Set `WORK_DIR` if not provided — default to `<document_stem>_wi/` alongside the document.
2. Create directories: `WORK_DIR/`, `WORK_DIR/images/`, `WORK_DIR/bulk/`.
3. Identify the document type from the file extension and, for PDFs, whether text is extractable:
   - **PDF** — check if text is readable:
     ```bash
     python3 -c "import fitz; doc = fitz.open('<DOCUMENT_PATH>'); print(doc[0].get_text()[:300])"
     ```
     - Output contains readable text → text-based PDF; proceed normally.
     - Output is empty → rasterized/scanned PDF; OCR will be required in Phase 2.
   - **DOCX / spreadsheet / other** — note the format; use appropriate libraries in Phase 2.
4. Report to the user: document type, page/sheet count, and whether text extraction is available.

---

## Phase 2: Extract Text

Extract all text content from the document so the agent can understand the process structure.

**PDF (text-based):**
```bash
python3 -c "
import fitz, json
doc = fitz.open('<DOCUMENT_PATH>')
pages = [{'page': i+1, 'text': doc[i].get_text()} for i in range(len(doc))]
print(json.dumps(pages, indent=2))
" > <WORK_DIR>/text_content.json
```

**PDF (rasterized — requires OCR):**
```bash
pip3 install pytesseract Pillow PyMuPDF  # if not already installed
brew install tesseract                   # if not already installed
python3 -c "
import fitz, pytesseract
from PIL import Image
import io, json
doc = fitz.open('<DOCUMENT_PATH>')
pages = []
for i in range(len(doc)):
    pix = doc[i].get_pixmap(dpi=200)
    img = Image.open(io.BytesIO(pix.tobytes('png')))
    text = pytesseract.image_to_string(img)
    pages.append({'page': i+1, 'text': text})
print(json.dumps(pages, indent=2))
" > <WORK_DIR>/text_content.json
```

**DOCX:**
```bash
pip3 install python-docx  # if not already installed
python3 -c "
from docx import Document
import json
doc = Document('<DOCUMENT_PATH>')
paragraphs = [p.text for p in doc.paragraphs if p.text.strip()]
print(json.dumps(paragraphs, indent=2))
" > <WORK_DIR>/text_content.json
```

Read `text_content.json` to understand the document structure before proceeding.

---

## Phase 3: Extract Images

Extract embedded images or diagrams from the document into `WORK_DIR/images/`.

**PDF (text-based) — extract embedded images:**

If PyMuPDF is not installed: `pip3 install PyMuPDF`

```bash
python3 -c "
import fitz, json
from pathlib import Path
doc = fitz.open('<DOCUMENT_PATH>')
images_dir = Path('<WORK_DIR>/images')
images_dir.mkdir(exist_ok=True)
manifest = []
for page_num in range(len(doc)):
    for img_idx, xref in enumerate(doc.get_page_images(page_num)):
        xref_id = xref[0]
        img_data = doc.extract_image(xref_id)
        filename = f'page_{page_num+1:02d}_img_{img_idx+1:02d}.{img_data[\"ext\"]}'
        (images_dir / filename).write_bytes(img_data['image'])
        manifest.append({'filename': filename, 'page_number': page_num+1})
(images_dir / 'manifest.json').write_text(json.dumps({'images': manifest}, indent=2))
print(f'{len(manifest)} images extracted')
"
```

**PDF (rasterized) — render full pages as images:**
```bash
python3 -c "
import fitz
from pathlib import Path
doc = fitz.open('<DOCUMENT_PATH>')
images_dir = Path('<WORK_DIR>/images')
images_dir.mkdir(exist_ok=True)
for i in range(len(doc)):
    pix = doc[i].get_pixmap(dpi=150)
    pix.save(str(images_dir / f'page_{i+1:02d}.png'))
print(f'{len(doc)} pages rendered')
"
```

**DOCX — extract embedded images:**
```bash
python3 -c "
from docx import Document
from pathlib import Path
import json
doc = Document('<DOCUMENT_PATH>')
images_dir = Path('<WORK_DIR>/images')
images_dir.mkdir(exist_ok=True)
manifest = []
for i, rel in enumerate(doc.part.rels.values()):
    if 'image' in rel.reltype:
        ext = rel.target_ref.split('.')[-1]
        filename = f'image_{i+1:02d}.{ext}'
        with open(images_dir / filename, 'wb') as f:
            f.write(rel.target_part.blob)
        manifest.append({'filename': filename})
(images_dir / 'manifest.json').write_text(json.dumps({'images': manifest}, indent=2))
print(f'{len(manifest)} images extracted')
"
```

Read the extracted images and report to the user: X images extracted.

---

## Phase 4: Analyze Text and Images to identify the process

Read the extracted text content and images (read all of them). From the document, identify:

- **What process is being demonstrated** — the product, the task, and its goal
- **Procedure boundaries** — natural breakpoints where one distinct sub-task ends and another begins (e.g., "Remove Lid" vs "Replace Lid")
- **Steps within each procedure** — the discrete actions described or shown
- **Key reference images** — images that best illustrate an important action or state; these will become visuals attached to the procedure

Ask the user to confirm your interpretation of the procedures and step breakdown before authoring the WI structure. Present a brief summary:

```
Proposed WI structure for <WI_NAME>:

  Procedure 1: <name>
    Step 1: <instruction> (work step)
    Step 2: <instruction> (check step)
    ...
    Reference images: image_01.png, image_02.png

  Procedure 2: <name>
    ...

Reply with any changes, or "ok" to proceed.
```

**Step type guidance:**
- `work_step` — an action the operator performs
- `check_step` — a verification or quality check (e.g., "Verify the lid is fully tightened and does not wobble")

---

## Phase 5: Select reference images

From the extracted images, choose the key images to attach as procedure visuals. Aim for:
- 1–3 images per procedure
- Images that show a distinct state (starting condition, key action, end result) rather than near-duplicates
- Ordered to match the process flow within the procedure

Copy selected images to `WORK_DIR/images/` with descriptive filenames if needed, and record the assignments in `WORK_DIR/image_assignments.json`:

```json
{
  "Procedure Name": ["image_01.png", "image_02.png"],
  "Procedure 2 Name": ["image_03.png"]
}
```

---

## Phase 6: Author the WI bulk-create JSON

Write `WORK_DIR/bulk/<wi-name-slug>.json` describing the full WI structure. This file is passed directly to `work-instruction:bulk-create`.

```json
{
  "name": "<WI_NAME>",
  "folder_uuid": "<FOLDER_UUID>",
  "procedures": [
    {
      "name": "Procedure Name",
      "safety_warning": null,
      "steps": [
        { "step_type": "work_step", "instruction": "Step text here." },
        { "step_type": "check_step", "instruction": "Verification step text." }
      ]
    }
  ]
}
```

Leave `folder_uuid` as a placeholder for now — it will be filled in after the folder is created in Phase 8a.

Show the user the full bulk JSON for review and confirm before creating anything in Threaded.

---

## Phase 7: Upload media

> **CLI only.** Uploading binary image files requires the Threaded CLI. The MCP server does not have access to your local filesystem.
>
> If you are running via MCP only, skip to Phase 8 to create the WI structure without visuals. You can attach visuals later using the CLI.

For each image in `WORK_DIR/images/`:

```bash
threaded task work-instruction:media:upload \
  --file "<WORK_DIR>/images/<filename>" \
  --organization <ORG_UUID>
```

Capture both `mediaUuid` and `cloudinaryPublicId` from each upload. Save a lookup map at `WORK_DIR/media_map.json`:

```json
{
  "<filename>.jpg": {
    "mediaUuid": "med-xxxx-xxxx-xxxx",
    "cloudinaryPublicId": "med-cloudinary-xxxx"
  }
}
```

Report progress (X of Y images uploaded).

---

## Phase 8: Create the folder and work instruction

### 8a. Create the folder

**MCP:**
```
execute_threaded_script(script="threaded task work-instruction:folder:create --name '<FOLDER_NAME>' --organization <ORG_UUID>")
```

**CLI:**
```bash
threaded task work-instruction:folder:create \
  --name "<FOLDER_NAME>" \
  --organization <ORG_UUID>
```

Capture the folder UUID from `{ "uuid": "wif-..." }`. Update the bulk JSON `folder_uuid` field with this value.

### 8b. Create the work instruction

Write the bulk JSON content to `WORK_DIR/bulk/<wi-name-slug>.json` (already done in Phase 6, updated with `folder_uuid`).

**MCP** — write the JSON to a temp file and run `bulk-create`:
```
execute_threaded_script(script="""cat > /tmp/<wi-name-slug>.json << 'EOJSON'
<contents of WORK_DIR/bulk/<wi-name-slug>.json>
EOJSON
threaded task work-instruction:bulk-create --file /tmp/<wi-name-slug>.json --organization <ORG_UUID>""")
```

**CLI:**
```bash
threaded task work-instruction:bulk-create \
  --file "<WORK_DIR>/bulk/<wi-name-slug>.json" \
  --organization <ORG_UUID>
```

The result includes `{ workInstructionVersionUuid, procedures: [{ name, uuid, stepCount }] }`.

Save the full result to `WORK_DIR/results.json`.

---

## Phase 9: Attach visuals

> **CLI only.** Attaching visuals requires media UUIDs from Phase 7, which requires the CLI. Skip this phase if running via MCP only.

For each procedure and each image assigned to it in `image_assignments.json`:

1. **Create a blank image visual:**
   ```bash
   threaded task work-instruction:visual:create-image \
     --procedure-version <procedure_uuid>
   ```
   Capture `{ "uuid": "piv-..." }`.

2. **Attach the image:**
   ```bash
   threaded task work-instruction:visual:update-image \
     --image-visual <visual_uuid> \
     --image-media-uuids '["<media_uuid>"]' \
     --remote-media-uuid <cloudinary_public_id> \
     --annotations '[{"className":"Image","attrs":{"id":"<generated-uuid>","name":"Image","mediaUuid":"<media_uuid>","x":0,"y":0,"width":500,"height":400,"scaleX":1,"scaleY":1,"draggable":true}}]'
   ```
   Look up `<media_uuid>` and `<cloudinary_public_id>` from `media_map.json`. Generate a fresh random UUID for the annotation `id` field (`import uuid; str(uuid.uuid4())`).

---

## Phase 10: Verify and summarize

1. Spot-check 1–2 procedures:

   **MCP:**
   ```
   execute_threaded_script(script="threaded task work-instruction:step:list --procedure-version <prv-uuid>")
   execute_threaded_script(script="threaded task work-instruction:visual:list --procedure-version <prv-uuid>")
   ```

   **CLI:**
   ```bash
   # Verify steps
   threaded task work-instruction:step:list --procedure-version <prv-uuid>

   # Verify visuals
   threaded task work-instruction:visual:list --procedure-version <prv-uuid>
   ```

   Confirm step count, step types, and (if visuals were attached) that each visual has a non-empty `remote_media_uuid`.

2. Print a summary:
   ```
   Import Complete
   ===============
   Document: <DOCUMENT_PATH>
   Work Instruction: <WI_NAME>
   Folder: <FOLDER_NAME> (<folder_uuid>)
   Procedures: N
   Steps: N total
   Images: N uploaded, N visuals created
   Results: WORK_DIR/results.json
   ```

---

## Tips for better results

**Specifying the WI structure**
- If you know the procedure breakdown in advance, tell the agent before Phase 4 (e.g., "Split into Remove Lid and Replace Lid, ~5 steps each")
- Specify which steps should be `check_step` vs `work_step` if you have quality-check requirements
