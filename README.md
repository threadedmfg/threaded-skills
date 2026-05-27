# threaded-skills

AI agent skills for working with Threaded. These skills teach AI agents (Cursor, Claude Code, and others) how to perform common Threaded workflows — importing data, building process maps, and creating work instructions from documents and videos.

## Skills

| Skill | Description |
|---|---|
| [create-tools-from-document](skills/create-tools-from-document/) | Import a tool inventory from a CSV, spreadsheet, or JSON file into Threaded |
| [create-parts-from-document](skills/create-parts-from-document/) | Import a part library from a CSV, spreadsheet, or JSON file into Threaded |
| [create-work-instructions-from-document](skills/create-work-instructions-from-document/) | Import work instructions from a PDF into Threaded — extracts images, structures procedures, and creates WIs with visuals attached |
| [create-work-instructions-from-video](skills/create-work-instructions-from-video/) | Create work instructions from a video demonstrating a process — extracts frames, identifies steps, and creates WIs with reference images |
| [create-map-from-document](skills/create-map-from-document/) | Build a Threaded Map (value stream map) from a process flow document |

## Installation

Skills are installed by copying the `skills/` directory contents into your AI agent's skills folder. Each agent looks for `SKILL.md` files one level deep inside the skills directory.

### Cursor

Copy skills to your project's `.cursor/skills/` folder, or to `~/.cursor/skills/` for personal (cross-project) access:

```bash
# Project-level (shared with collaborators via version control)
cp -r skills/* /path/to/your/project/.cursor/skills/

# Personal (available across all your projects)
cp -r skills/* ~/.cursor/skills/
```

### Claude Code

Copy skills to your project's `.claude/skills/` folder:

```bash
cp -r skills/* /path/to/your/project/.claude/skills/
```

After copying, the agent will automatically discover and use the skills when relevant tasks are requested.

## Prerequisites

All skills require the Threaded CLI to be installed and authenticated. Run `threaded auth status` to verify.

| Skill | Additional requirements |
|---|---|
| create-tools-from-document | Python 3; `pip3 install openpyxl` for XLSX sources |
| create-parts-from-document | Python 3; `pip3 install openpyxl` for XLSX sources |
| create-work-instructions-from-document | Python 3; `pip3 install PyMuPDF`; for rasterized PDFs also `brew install tesseract` and `pip3 install pytesseract Pillow numpy scipy` |
| create-work-instructions-from-video | `brew install ffmpeg`; Python 3 |
| create-map-from-document | Python 3; `pip3 install openpyxl` for XLSX sources |
