# threaded-skills

AI agent skills for working with Threaded. These skills teach AI agents (Cursor, Claude Code, and others) how to perform common Threaded workflows — importing data and creating work instructions from documents and videos.

## Skills

| Skill | Description | MCP | CLI |
|---|---|---|---|
| [create-tools-from-document](skills/create-tools-from-document/) | Import a tool inventory from a CSV, spreadsheet, or JSON file into Threaded | ✓ | ✓ |
| [create-parts-from-document](skills/create-parts-from-document/) | Import a part library from a CSV, spreadsheet, or JSON file into Threaded | ✓ | ✓ |
| [create-work-instructions-from-document](skills/create-work-instructions-from-document/) | Import work instructions from a document — extracts images, structures procedures, and creates WIs with visuals attached | Partial* | ✓ |
| [create-work-instructions-from-video](skills/create-work-instructions-from-video/) | Create work instructions from a video demonstrating a process — extracts frames, identifies steps, and creates WIs with reference images | — | ✓ |

\* MCP supports creating folders and WI structure. Media upload and visual attachment require the CLI.

## Before You Start

### 1. Connect your AI agent to Threaded

Skills work with the **Threaded MCP server** or the **Threaded CLI**. The MCP server is the easiest way to get started and requires no local installation.

**MCP (recommended for most users)**

Add the Threaded MCP server to your agent. Instructions are in your Threaded account under **Settings → Integrations → MCP**. Once connected, the agent will detect the MCP runtime automatically and no additional software is needed for tools and parts imports.

**CLI**

Install the Threaded CLI from [threadedmfg.com/docs/cli](https://threadedmfg.com/docs/cli), then authenticate:

```bash
threaded auth login
threaded auth status  # confirm you are logged in
```

### 2. Find your Organization UUID

Most Threaded commands require your organization UUID. To find it, run:

**MCP:**
```
execute_threaded_script(script="threaded auth orgs")
```

**CLI:**
```bash
threaded auth orgs
```

Copy the UUID from the output — it looks like `org-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`. You will be asked for this when starting any skill.

### 3. Choose MCP or CLI

| | MCP | CLI |
|---|---|---|
| Tools and parts imports | ✓ Full support | ✓ Full support |
| Work instruction structure (folders, procedures, steps) | ✓ Full support | ✓ Full support |
| Media upload and visual attachment | — Not supported | ✓ Required |
| Setup required | None (add MCP server) | Install + authenticate CLI |

If you are importing work instructions **with images**, you need the CLI for the upload and attachment steps, even if you use MCP for everything else.

## Installation

### Via npx (recommended)

Use [`npx skills`](https://github.com/vercel-labs/skills) to install to Cursor, Claude Code, and [50+ other agents](https://github.com/vercel-labs/skills#supported-agents):

```bash
# Install all skills to your project
npx skills add threadedmfg/threaded-skills

# Install globally (available across all your projects)
npx skills add threadedmfg/threaded-skills --global

# Install a specific skill
npx skills add threadedmfg/threaded-skills --skill create-work-instructions-from-document

# Install to a specific agent
npx skills add threadedmfg/threaded-skills --agent cursor
npx skills add threadedmfg/threaded-skills --agent claude-code
```

### Manual install

Copy skills directly into your agent's skills folder:

```bash
# Cursor — project-level (shared with team via version control)
cp -r skills/* /path/to/your/project/.cursor/skills/

# Cursor — personal (available across all your projects)
cp -r skills/* ~/.cursor/skills/

# Claude Code
cp -r skills/* /path/to/your/project/.claude/skills/
```

After installing, the agent will automatically discover and use the skills when relevant tasks are requested.

## Quick Start

Here is a typical first import using the `create-tools-from-document` skill:

1. Install the skills and connect your agent to Threaded (see above).
2. Place your tool inventory file (CSV, XLSX, or JSON) somewhere the agent can read it.
3. Ask the agent:
   > "Import my tool inventory from `tool_list.csv` into Threaded."
4. The agent will:
   - Read and display the file's columns
   - Propose a field mapping and wait for your confirmation
   - Run a dry run and show a preview — **no data is written yet**
   - Ask you to confirm before creating any records
5. Review the preview, approve or adjust the mapping, and the import runs.

The same pattern applies to `create-parts-from-document`. Work instruction skills follow the same confirm-before-write flow but involve more phases.

## Prerequisites

| Skill | Additional requirements |
|---|---|
| create-tools-from-document | Python 3; `pip3 install openpyxl` for XLSX sources |
| create-parts-from-document | Python 3; `pip3 install openpyxl` for XLSX sources |
| create-work-instructions-from-document | Python 3; `pip3 install PyMuPDF`; for rasterized PDFs also `brew install tesseract` and `pip3 install pytesseract Pillow` |
| create-work-instructions-from-video | `brew install ffmpeg`; Python 3 |

## Data Privacy

Source documents, videos, extracted text, video frames, and intermediate JSON files may contain sensitive manufacturing data. Before running any skill:

- Confirm that the AI agent you are using is approved to process the source files.
- Use a work directory (`WORK_DIR`) on storage you control.
- Review extracted text and the proposed import JSON before confirming the import.
- Clean up intermediate files in `WORK_DIR` after import if you do not want them retained on disk.

## Troubleshooting

**`threaded: command not found`**
The Threaded CLI is not installed or not on your PATH. See the [CLI installation docs](https://threadedmfg.com/docs/cli).

**`Not authenticated` or `401` errors**
Run `threaded auth login` and follow the prompts, then re-run `threaded auth status` to confirm.

**`No MCP tools found` / agent does not detect Threaded**
The Threaded MCP server is not connected to your agent. Check **Settings → Integrations → MCP** in your Threaded account for setup instructions.

**`ModuleNotFoundError: No module named 'fitz'`**
Run `pip3 install PyMuPDF`.

**`ModuleNotFoundError: No module named 'openpyxl'`**
Run `pip3 install openpyxl`.

**`ffmpeg: command not found`**
Run `brew install ffmpeg` (macOS) or follow [ffmpeg.org/download.html](https://ffmpeg.org/download.html) for other platforms.

**Media upload steps are skipped or fail**
Uploading images and video frames requires the Threaded CLI. If you are running via MCP only, the skill will complete the WI structure but cannot upload media. Switch to the CLI for those phases or run the upload steps manually.

**Import created duplicate records**
By default the CLI warns on duplicate names but still creates the record. Use `--skip-existing` on the next run to skip any already-imported records. There is no bulk-delete command; duplicates must be removed through the Threaded app UI.
