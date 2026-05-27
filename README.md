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

## Prerequisites

Skills work with the **Threaded MCP server** or the **Threaded CLI**. The MCP server is the easiest way to get started — no local installation required.

### MCP users

Connect your AI agent to the Threaded MCP server and authenticate. Once connected, skills will detect the MCP runtime automatically. No additional software is needed for tools and parts imports.

### CLI users

Install and authenticate the Threaded CLI:

```bash
threaded auth status  # verify you're logged in
```

| Skill | Additional requirements |
|---|---|
| create-tools-from-document | Python 3; `pip3 install openpyxl` for XLSX sources |
| create-parts-from-document | Python 3; `pip3 install openpyxl` for XLSX sources |
| create-work-instructions-from-document | Python 3; `pip3 install PyMuPDF`; for rasterized PDFs also `brew install tesseract` and `pip3 install pytesseract Pillow` |
| create-work-instructions-from-video | `brew install ffmpeg`; Python 3 |
