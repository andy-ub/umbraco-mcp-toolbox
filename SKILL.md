---
name: mcp-toolbox-sqlserver
description: Use this skill to set up Google's MCP Toolbox for direct SQL Server database access in Claude Code. Invoke whenever the user wants to connect Claude to a SQL Server (MSSQL) database, configure mcp-toolbox, create a tools.yaml, or enable natural-language database queries via MCP tools. Also invoke if the user mentions setting up database tools, wants to query Umbraco Engage data, or wants to avoid writing manual PowerShell/SQL queries during development.
---

# MCP Toolbox — SQL Server Setup

> **Platform:** Windows only (v1.0). macOS/Linux support is not yet covered — binary paths and shell commands below are PowerShell-specific.

Sets up [Google's MCP Toolbox for Databases](https://github.com/googleapis/mcp-toolbox) to give Claude Code direct SQL Server query access via MCP tools — so instead of writing PowerShell queries by hand, you call named tools like `mcp__myapp_db__list_tables` directly from Claude.

## What you'll have at the end

- `toolbox.exe` binary stored in `~/.claude/mcp-toolbox/`
- `tools.yaml` with your SQL Server connection + auto-discovered query tools
- MCP server registered in Claude Code (user-scoped, available in all projects)
- Tools callable directly from Claude: `mcp__<name>__list_tables`, `mcp__<name>__describe_table`, and any business tools you add

---

## Step 1 — Collect connection details

Ask the user for the following. Show the default or sample in brackets so they know what format to use:

| Field | Default / Sample |
|---|---|
| Host | `localhost` |
| Port | `1433` |
| Database | `UmbracoEngageV17_TestSite` |
| User | `sa` |
| Password | *(required — no default; example format: `YourPassword@123`)* |
| MCP server name | Based on database name, e.g. `myapp_db` — **must use underscores, not hyphens** (see pitfall #1) |

---

## Step 2 — Check / download toolbox binary

Check if the binary already exists:

```powershell
Test-Path "$env:USERPROFILE\.claude\mcp-toolbox\toolbox.exe"
```

If not found, download it (~271MB, takes 1–5 minutes). Check [the latest release](https://github.com/googleapis/mcp-toolbox/releases/latest) first and substitute the version if a newer one is available:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\mcp-toolbox" | Out-Null
$url = "https://storage.googleapis.com/mcp-toolbox-for-databases/v1.1.0/windows/amd64/toolbox.exe"
$dest = "$env:USERPROFILE\.claude\mcp-toolbox\toolbox.exe"
Invoke-WebRequest -Uri $url -OutFile $dest -UseBasicParsing
```

Verify it runs:

```powershell
& "$env:USERPROFILE\.claude\mcp-toolbox\toolbox.exe" --version
# Expected: toolbox version 1.1.0+binary.windows.amd64.* (or newer)
```

---

## Step 3 — Create tools.yaml (discovery tools first)

The golden rule: **never guess table or column names**. Create tools.yaml with just the source config and two discovery tools. You'll add business-specific tools in Step 5 after seeing the real schema.

Save to `$env:USERPROFILE\.claude\mcp-toolbox\tools.yaml`:

```yaml
sources:
  <server_name>:
    kind: mssql
    host: <host>
    port: <port>
    database: <database>
    user: <user>
    password: "<password>"
    encrypt: disable        # required for localhost without a proper TLS cert

tools:
  list_tables:
    kind: mssql-sql
    source: <server_name>
    description: List all user tables in the database, grouped by schema. Run this first to discover real table names before writing any queries.
    statement: >
      SELECT TABLE_SCHEMA, TABLE_NAME
      FROM INFORMATION_SCHEMA.TABLES
      WHERE TABLE_TYPE = 'BASE TABLE'
      ORDER BY TABLE_SCHEMA, TABLE_NAME

  describe_table:
    kind: mssql-sql
    source: <server_name>
    description: Show all columns and data types for a given table. Run this before writing any SQL that references specific columns.
    parameters:
      - name: table_name
        type: string
        description: Name of the table to describe (e.g. umbracoEngagePersonalizationCampaignGroup)
    statement: >
      SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, CHARACTER_MAXIMUM_LENGTH
      FROM INFORMATION_SCHEMA.COLUMNS
      WHERE TABLE_NAME = @table_name
      ORDER BY ORDINAL_POSITION

toolsets:
  default:
    description: All configured tools
    tools:
      - list_tables
      - describe_table
```

---

## Step 4 — Register MCP server

```powershell
$binary = "$env:USERPROFILE\.claude\mcp-toolbox\toolbox.exe"
$config = "$env:USERPROFILE\.claude\mcp-toolbox\tools.yaml"
claude mcp add --scope user <server_name> -- $binary --config $config --stdio
```

Verify the server is connected:

```powershell
claude mcp list
# Should show: <server_name>: ... ✓ Connected
```

> **⚠️ Pitfall #1 — Server name must use underscores**: Claude Code's VS Code extension will NOT expose tools from a server whose name contains hyphens. `engage-db` → tools invisible. `engage_db` → tools visible. Always name servers with underscores.

**Then ask the user to restart VS Code** (close and reopen the window completely). The MCP tool list is only loaded when a session starts — without a restart, the new tools won't appear in the current session.

---

## Step 5 — Schema-first: discover, then write business tools

After the user has restarted VS Code and confirmed the tools are available:

### 5a — Discover table names

Call `mcp__<server_name>__list_tables` (no parameters). Review the output with the user — identify which tables are relevant to what they want to query.

### 5b — Discover column names

For each relevant table, call `mcp__<server_name>__describe_table` with `{"table_name": "<table>"}`. Note the real column names and types.

### 5c — Write business tools

Add tools to `tools.yaml` based on the actual schema. Example:

```yaml
  get_campaign_groups:
    kind: mssql-sql
    source: <server_name>
    description: List all campaign groups with their associated campaigns and UTM parameters.
    statement: >
      SELECT cg.id, cg.name, cg.description,
             c.id AS campaignId, c.utmSource, c.utmMedium, c.utmCampaign
      FROM umbracoEngagePersonalizationCampaignGroup cg
      LEFT JOIN umbracoEngagePersonalizationCampaign c ON c.campaignGroupId = cg.id
      ORDER BY cg.name
```

Add each new tool to the `toolsets.default.tools` list too.

> **⚠️ Pitfall #2 — SQL Server reserved words**: Some common words are reserved and will cause a syntax error when used as column aliases without quoting. Wrap them: `p.rows AS [RowCount]`, `t.[key]`, `t.[name]`.

Test each new tool by calling it directly from Claude (e.g. `mcp__<server_name>__get_campaign_groups`) to confirm the SQL is correct before considering the setup done.

---

## Step 6 — Verify end-to-end

Call one business tool. Confirm the result contains real data from the database. If a tool fails:

- **"Invalid column name"** → run `describe_table` to check the real column names
- **"Invalid object name"** → run `list_tables` to check the real table name
- **Syntax error near reserved word** → add `[brackets]` around the alias
- **Connection refused** → check that the DB container is running and port is reachable
- **Tools not visible after registration** → restart VS Code

---

## Known pitfalls summary

| # | Symptom | Cause | Fix |
|---|---------|-------|-----|
| 1 | MCP tools not visible in session | Hyphen in server name | Rename server with underscore, restart VS Code |
| 2 | SQL syntax error | Reserved word used as alias | Wrap alias with `[brackets]` |
| 3 | "Invalid object name" | Guessed wrong table name | Run `list_tables` first |
| 4 | "Invalid column name" | Guessed wrong column name | Run `describe_table` first |
| 5 | Tools invisible after registration | VS Code not restarted | Close and reopen VS Code |
| 6 | `port 5000 already in use` in logs | HTTP mode conflict (harmless) | Ignore — stdio mode doesn't use port 5000 |
| 7 | TLS/certificate error | Missing `encrypt: disable` | Add `encrypt: disable` to source config |
