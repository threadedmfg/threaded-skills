---
name: threaded-runner-setup
description: >-
  Detect available Threaded runtime (MCP server or local CLI) and run commands
  in the correct mode. Read this before starting any Threaded skill.
---

# Threaded Runner Setup

Read this file at the start of any Threaded skill to determine how to invoke Threaded commands.

---

## Step 1 — Detect your runtime

**MCP (Threaded MCP Server)**
You have access to the `execute_threaded_script` and `threaded_auth_status` tools. No local CLI installation is needed.

**CLI (Threaded CLI)**
You have the `threaded` command available in your terminal, installed and authenticated locally.

If both are available, prefer the MCP server.

---

## Step 2 — Check auth status

Verify you are authenticated before running any commands.

**MCP:** Call the `threaded_auth_status` tool (no arguments). Confirm the result shows an active session.

**CLI:**
```bash
threaded auth status
```

---

## Step 3 — Confirm the organization UUID

Most Threaded commands require `--organization <ORG_UUID>`. Ask the user for their organization UUID if not already provided.

To list available organizations:

**MCP:**
```
execute_threaded_script(script="threaded auth orgs")
```

**CLI:**
```bash
threaded auth orgs
```

---

## Step 4 — Running Threaded commands

All `threaded task ...` command strings are **identical** in both modes. Only the invocation differs.

**MCP** — pass the full command string as the `script` argument:
```
execute_threaded_script(script="threaded task tool:list --organization <ORG_UUID>")
```

**CLI** — run directly in your terminal:
```bash
threaded task tool:list --organization <ORG_UUID>
```

---

## Step 5 — File operations

**Structured data (JSON):** Use inline flags (`--tools '<json>'`, `--parts '<json>'`) to pass JSON arrays directly in the command string. This works in both MCP and CLI modes and avoids intermediate files.

Wrap the JSON in single quotes in the command string. Example:
```
threaded task tool:bulk-create --tools '[{"name":"Torque Wrench","cost":"45.00"}]' --organization <ORG_UUID>
```

For large datasets (100+ records), split into batches of ~50 and run multiple commands. The `--skip-existing` flag makes re-runs safe.

**Binary files (images, video):** Uploading binary files requires the CLI. The MCP server does not have access to your local filesystem. Skills that upload images or video are **CLI only** for those steps.
