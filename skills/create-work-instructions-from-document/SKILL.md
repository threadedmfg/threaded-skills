---
name: create-work-instructions-from-document
description: >-
  Create work instructions in Threaded from a document showing a manufacturing
  or assembly process. Extracts images from the document, analyzes images and text
  to identify procedures and steps, authors the work instruction structure, and imports it
  into Threaded with reference images attached as visuals. Use when a user has a
  document demonstrating a process and wants to turn it into Threaded work
  instructions.
---

# Create Work Instructions from a Document

## Collect inputs before starting

Ask the user for these if not already provided:

- **`ORG_UUID`** (required) — Threaded organization UUID
- **`VIDEO_PATH`** (required) — absolute or relative path to the video file (`.mp4`, `.mov`, `.avi`, etc.)
- **`WORK_DIR`** (optional) — directory to write intermediate files; defaults to a folder named after the video stem alongside the video file
- **`WI_NAME`** (optional) — name for the work instruction; if not provided, infer from the video content after inspecting it
- **`FOLDER_NAME`** (optional) — name for the WI folder in Threaded; defaults to `WI_NAME` if not provided

---

## Phase 1: Inspect the Document

## Phase 2: Extract Text

## Phase 3: Extract Images

## Phase 4: Analyze Text and Images to identify the process

Read the extracted frames as images (read all of them, in order). From the frame sequence, identify:

- **What process is being demonstrated** — the product, the task, and its goal
- **Procedure boundaries** — natural breakpoints where one distinct sub-task ends and another begins (e.g., "Remove Lid" vs "Replace Lid")
- **Steps within each procedure** — the discrete actions visible in the frames
- **Key reference frames** — frames that best illustrate an important action or state; these will become visuals attached to the procedure

Ask the user to confirm your interpretation of the procedures and step breakdown before authoring the WI structure. Present a brief summary:

```
Proposed WI structure for <WI_NAME>:

  Procedure 1: <name>
    Step 1: <instruction> (work step)
    Step 2: <instruction> (check step)
    ...
    Reference images: frame_0002.jpg, frame_0004.jpg

  Procedure 2: <name>
    ...

Reply with any changes, or "ok" to proceed.
```

**Step type guidance:**
- `work_step` — an action the operator performs
- `check_step` — a verification or quality check (e.g., "Verify the lid is fully tightened and does not wobble")

---

## Phase 5: Select reference images


## Phase 6: Author the WI bulk-create JSON

Write `WORK_DIR/bulk/<wi-name-slug>.json` describing the full WI structure. This file is passed directly to `work-instruction:bulk-create`.

## Phase 7: Upload media

### Calling the Threaded CLI

> **Important:** `threaded` is a shell function, not a standalone binary. It must be called from a shell where the Threaded CLI has been sourced.
>
> From a terminal (normal usage):
> ```bash
> threaded task work-instruction:media:upload ...
> ```
>
> From a Python subprocess, use `zsh -i -c`:
> ```python
> import subprocess, json
>
> THREADED_CLI_PATH = "<path to your Threaded CLI shell aliases file>"
>
> def run_threaded(cmd_args: str):
>     result = subprocess.run(
>         ["zsh", "-i", "-c", f"source {THREADED_CLI_PATH}; threaded {cmd_args}"],
>         capture_output=True, text=True,
>     )
>     output = result.stdout
>     idx_obj = output.find('{')
>     idx_arr = output.find('[')
>     candidates = [i for i in [idx_obj, idx_arr] if i != -1]
>     if not candidates:
>         raise RuntimeError(f"No JSON in output (exit {result.returncode}):\n{output[:400]}\nstderr:\n{result.stderr[:400]}")
>     return json.loads(output[min(candidates):])
> ```

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

```bash
threaded task work-instruction:folder:create \
  --name "<FOLDER_NAME>" \
  --organization <ORG_UUID>
```

Capture the folder UUID from `{ "uuid": "wif-..." }`. Update the bulk JSON `folder_uuid` field with this value.

### 8b. Create the work instruction

```bash
threaded task work-instruction:bulk-create \
  --file "<WORK_DIR>/bulk/<wi-name-slug>.json" \
  --organization <ORG_UUID>
```

The result includes `{ workInstructionVersionUuid, procedures: [{ name, uuid, stepCount }] }`.

Save the full result to `WORK_DIR/results.json`.

---

## Phase 9: Attach visuals

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
   ```bash
   # Verify steps
   threaded task work-instruction:step:list --procedure-version <prv-uuid>

   # Verify visuals
   threaded task work-instruction:visual:list --procedure-version <prv-uuid>
   ```
   Confirm step count, step types, and that each visual has a non-empty `remote_media_uuid`.

2. Print a summary:
   ```
   Import Complete
   ===============
   Video: <VIDEO_PATH>
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
- If you know the procedure breakdown in advance, tell the agent before Phase 3 (e.g., "Split into Remove Lid and Replace Lid, ~5 steps each")
- Specify which steps should be `check_step` vs `work_step` if you have quality-check requirements
