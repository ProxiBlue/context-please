---
name: sync-vector-db
description: Sync or initialize vector DB indexes for grouped vendor modules (Mage-OS, Hyvä, etc.). Reads group definitions from .claude/vector-index-groups.json and calls context-please sync_codebase for each module. Supports per-group vendor_root overrides for multi-vendor indexing.
---

This skill syncs vector DB indexes for vendor modules using the context-please MCP server.

## What This Skill Does

Reads `.claude/vector-index-groups.json` from the project root, resolves module paths under vendor directories, and calls `context-please/sync_codebase` for each module. Each group can optionally specify its own `vendor_root` to support modules from different vendors (e.g., `vendor/mage-os/`, `vendor/hyva-themes/`). Already-indexed modules get incremental sync (detecting added/removed/modified files). Unindexed modules get full initial indexing.

## Arguments

The skill accepts optional arguments after invocation:

- **No arguments**: Sync all groups
- **`--group <name>`**: Sync only the named group (e.g., `--group MageOS-Catalog`)
- **`--list`**: List all groups and their modules without syncing
- **`--dry-run`**: Show what would be synced without calling MCP
- **`--health`**: Run Milvus health checks only (no syncing). Reports server status, collection load states, and any unloaded collections.
- **`--health --verbose`**: Full health report including per-collection details (load state, index info, row counts)

## Config File

`$PROJECT_ROOT/.claude/vector-index-groups.json` with structure:

```json
{
  "base_path": "/var/www/html",
  "vendor_root": "vendor/mage-os",
  "parallel": 2,
  "groups": [
    { "name": "GroupName", "modules": ["module-foo", "module-bar"] },
    { "name": "PatternGroup", "patterns": ["module-inventory*"] },
    { "name": "Everything-Else", "remaining": true },
    { "name": "Hyva", "vendor_root": "vendor/hyva-themes", "modules": ["hyva-ui", "magento2-theme-module"] }
  ]
}
```

**Per-group `vendor_root`:** If a group defines its own `vendor_root`, use that instead of the top-level `vendor_root` when constructing paths for that group's modules. This allows indexing modules from multiple vendors (e.g., Mage-OS + Hyvä) in a single config.

## Execution Instructions

When this skill is invoked, follow these steps exactly:

### Step 1: Read the config

Read `.claude/vector-index-groups.json` from the project root. Extract `base_path`, `vendor_root`, and `groups`.

Also read `.mcp.json` from the project root. Extract the Milvus connection from `mcpServers.context-please.env.MILVUS_ADDRESS`. This value is `host:port` (e.g., `167.99.36.131:19530`). Derive:
- **REST API:** `http://<MILVUS_ADDRESS>` (the address already includes the REST API port, typically 19530)
- **Management API:** `http://<host>:9091` (replace the port with 9091, same host)

If `context-please` or `MILVUS_ADDRESS` is not found in `.mcp.json`, health checks are unavailable.

### Step 2: Parse arguments

Check if the user passed arguments (after `/sync-vector-db`). Parse `--group`, `--list`, `--dry-run`, `--health`, `--verbose` flags.

### Step 3: If --health, run Milvus health checks and stop

Use the Milvus endpoints derived from `.mcp.json` in Step 1. If `MILVUS_ADDRESS` was not found, report that Milvus endpoints are not configured in `.mcp.json` and stop.

Run these checks using `curl` via Bash:

**3a. Server health:**
```bash
curl -s <management_api>/api/v1/health
```
Expected: `{"status":"ok"}`. If unreachable, report Milvus is down and stop.

**3b. List all collections:**
```bash
curl -s <rest_api>/v2/vectordb/collections/list -X POST -H 'Content-Type: application/json' -d '{}'
```
Report total collection count.

**3c. Check load state of all collections:**

Write a Python script to the scratchpad directory that:
1. Reads the collection list from stdin (piped from 3b)
2. For each collection, calls `<rest_api>/v2/vectordb/collections/get_load_state`
3. Reports: total loaded, total not loaded, and lists any not-loaded collections with their state

```bash
curl -s <rest_api>/v2/vectordb/collections/list -X POST -H 'Content-Type: application/json' -d '{}' | python3 <scratchpad>/check_milvus.py
```

**3d. If --verbose, get details per collection:**

For each collection, call:
```bash
curl -s <rest_api>/v2/vectordb/collections/describe -X POST -H 'Content-Type: application/json' -d '{"collectionName": "<name>"}'
```

Report per collection: name, description (contains module name), load state, index types, row count.

**3e. Report summary:**
```
Milvus Health Report
  Server: <management_api> -- OK/UNREACHABLE
  Collections: X total, Y loaded, Z not loaded
  [If not loaded] Unloaded collections: <list>
```

After reporting, stop. Do not proceed to sync.

### Step 4: Check MCP tool schema

Run: `mcp-cli info context-please/sync_codebase`

### Step 5: Resolve vendor_root and modules per group

For each group (or the filtered group if `--group` was specified):

1. **Resolve vendor_root:** If the group has its own `vendor_root` property, use `{base_path}/{group.vendor_root}` as the vendor directory for that group. Otherwise fall back to `{base_path}/{top-level vendor_root}`.

2. **Resolve modules:**
    - **`modules` array**: Use each entry directly. Verify `{vendor_dir}/{module}` exists.
    - **`patterns` array**: Glob-expand each pattern against the group's resolved vendor directory.
    - **`remaining: true`**: Collect all `module-*` directories in the **top-level** vendor dir that are NOT claimed by any other group. (Remaining only applies to the default vendor_root, not per-group overrides.)

### Step 6: If --list, display and stop

Print each group name with its module count and module names.

### Step 7: If --dry-run, display and stop

Print each group and what would be synced.

### Step 8: Pre-sync Milvus health check

Before syncing, if Milvus endpoints were resolved from `.mcp.json`, run a quick health check (Step 3a only -- server reachability). If Milvus is unreachable, warn the user and stop. Do not attempt to sync against a dead Milvus instance.

### Step 9: Sync each module

**IMPORTANT: Process modules strictly one at a time (sequential only). Do NOT run multiple mcp-cli calls in parallel.** The Milvus vector DB cannot handle concurrent writes reliably and will fail or corrupt data under parallel load. Wait for each module's sync to complete before starting the next.

**CRITICAL: Each `mcp-cli call` MUST be a separate Bash tool invocation.** Do NOT write a bash loop script that calls `mcp-cli` multiple times in a single Bash command. The MCP server maintains in-memory state (including the snapshot manifest) across calls made through Claude Code's Bash tool, but a standalone bash script spawns separate short-lived MCP server processes for each `mcp-cli` call. Those separate processes each load the snapshot from disk, update only one entry, and exit -- causing writes from previous iterations to be overwritten. The result is a snapshot with far fewer entries than expected. Always use individual Bash tool calls for each module.

For each module in each group, call the MCP tool via a **separate Bash tool invocation**:

```
mcp-cli call context-please/sync_codebase '{"path": "<vendor_dir>/<module_name>", "base_path": "<base_path>", "force": true, "blocking": true}'
```

Where:
- `<vendor_dir>` = `<base_path>/<group.vendor_root or top-level vendor_root>` (e.g., `/var/www/html/vendor/mage-os` or `/var/www/html/vendor/hyva-themes`)
- `<module_name>` = the module directory name (e.g., `module-catalog` or `hyva-ui`)
- `<base_path>` = from config (e.g., `/var/www/html`)
- `force: true` means unindexed modules get initial indexing automatically
- `blocking: true` waits for indexing to fully complete before returning, returning final stats (files indexed, chunks inserted) instead of a "started background indexing" acknowledgement. Prevents Milvus overload during sequential bulk indexing.

### Step 10: Report results

Parse each response. The response JSON has structure `{"content": [{"type": "text", "text": "..."}], "isError": boolean}`.

Classify results:
- "no changes" in text = already synced, no updates needed
- "Sync complete" with Added/Removed/Modified counts = incremental sync done
- "Indexing complete" with files/chunks stats = full index completed (blocking mode)
- "Indexing failed" = indexing error (blocking mode)
- `isError: true` = failure

Report a summary per group: `[GroupName] X ok, Y failed`

### Step 11: Final summary

Report total modules processed, successes, failures.

### Step 12: Verify snapshot

After all syncs complete, verify the snapshot file has the expected number of entries:

```bash
python3 -c "import json; d=json.load(open('<base_path>/.context/mcp-codebase-snapshot.json')); print(f'Snapshot entries: {len(d.get(\"codebases\",{}))}')"
```

The snapshot count should be close to the total modules synced. If it is significantly lower (e.g., less than 80% of total modules), the sync was likely run via a bash script loop instead of individual Bash tool calls -- warn the user and advise rerunning.

## Important Notes

- This skill MUST run inside Claude Code where MCP servers are connected
- The `force: true` parameter ensures unindexed modules get initial indexing rather than an error
- `base_path` is passed so collection naming uses portable keys (relative paths), critical for cross-environment compatibility
- Groups with `"remaining": true` act as catch-all for unclaimed modules in the top-level vendor_root only
- Groups with a per-group `vendor_root` override use that path instead of the top-level `vendor_root`
- Process ALL modules strictly sequentially - one mcp-cli call at a time, never parallel. Milvus DB overloads under concurrent writes
- **Never use bash loop scripts for mcp-cli calls** - each call must be a separate Bash tool invocation to preserve MCP server in-memory state
