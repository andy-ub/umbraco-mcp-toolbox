# mcp-toolbox-sqlserver

A Claude Code skill that connects Claude directly to your SQL Server database during development.

When building database-backed tools, you constantly need to verify what data is in the database — to confirm test data exists before smoke testing, explore table schemas before writing queries, or debug unexpected results against raw records. This skill removes the PowerShell step: Claude queries the database on your behalf, inline in the conversation where you're already working.

## What it does

1. Downloads the `toolbox.exe` binary if not already present
2. Collects your SQL Server connection details (with sensible defaults)
3. Creates a `tools.yaml` following a **schema-first workflow** — discovers real table/column names before writing any queries
4. Registers the MCP server in Claude Code
5. Guides you through VS Code restart and verification
6. Flags known pitfalls so you don't hit them

## How to trigger it

Invoke with `/mcp-toolbox-sqlserver`, or just describe what you want — Claude will pick it up:

```
# Direct invocation
/mcp-toolbox-sqlserver

# Natural language — all of these trigger the skill:
"setup mcp toolbox for my SQL Server database"
"connect Claude Code to my MSSQL dev database without PowerShell"
"I want to query Umbraco Engage data directly from Claude"
"help me setup database tools for Claude Code, SQL Server on localhost:1433"
```

## Sample onboarding prompts

Use these as starting points. Claude will ask for any missing details.

> **Security note:** Never paste real production passwords into a chat prompt. Use these patterns with dev/local credentials only.

**Minimal — just tell it the DB:**
```
setup mcp toolbox for SQL Server, database UmbracoEngageV17_TestSite on localhost
```

**With full credentials (dev environment):**
```
I want to setup mcp toolbox to query my SQL Server database directly from Claude Code.
Server: localhost, port: 1433, database: UmbracoEngageV17_TestSite, user: sa
```

**From a fresh Umbraco Engage clone:**
```
I just cloned the Umbraco Engage repo and got the TestSite running.
I want Claude to be able to check whether reporting tables are populated,
explore the schema before I write queries, and spot-check data while I implement tools.
How do I connect Claude to the database?
```

**Generic project (non-Umbraco):**
```
help me connect Claude Code to my MSSQL dev database so I can query it without writing PowerShell.
SQL Server in a Docker container on localhost:1433, database MyAppDb, user dev
```

## Defaults

| Field | Default / Sample |
|---|---|
| Host | `localhost` |
| Port | `1433` |
| Database | `UmbracoEngageV17_TestSite` |
| User | `sa` |
| Password | *(required — example: `YourPassword@123`)* |
| MCP server name | Based on DB name, e.g. `myapp_db` (underscores only) |

## Benchmark results (iteration 1, 2026-05-07)

Evaluated against 3 real-world prompts, each run with and without the skill.

| Eval | with_skill | without_skill |
|---|---|---|
| 1 — Umbraco Engage, standard setup | 8/8 ✅ | 4/8 |
| 2 — Docker MSSQL, generic project | 8/8 ✅ | 4/8 |
| 3 — Umbraco natural language query | 8/8 ✅ | 5/8 ⚠️ |
| **Pass rate** | **100%** | **48%** |

| Metric | with_skill | without_skill |
|---|---|---|
| Avg tokens | 30,450 | 22,126 (+38% overhead) |
| Avg duration | 160s | 103s (+55% overhead) |

**What the skill adds over baseline:**
- Correct `kind: mssql` source type (baseline used `cloud-sql-mssql` or Docker HTTP mode)
- Schema-first workflow enforced (baseline wrote all tools upfront, guessing table/column names)
- Underscore naming explained explicitly (baseline got it right by accident or missed it)
- Correct `claude mcp add` flags (baseline used wrong flags or wrong transport)
- `encrypt: disable` for localhost (baseline often missed this)

**Known eval limitation:** Eval 3 baseline scored slightly higher (5/8) because the subagent could see the existing `toolbox.exe` on disk, leaking context. A clean-machine baseline would likely score 3-4/8.

## Known pitfalls (encoded in skill)

| # | Symptom | Fix |
|---|---------|-----|
| 1 | MCP tools not visible after setup | Server name has hyphen → rename with underscore, restart VS Code |
| 2 | SQL syntax error on alias | Reserved word → wrap with `[brackets]` |
| 3 | "Invalid object name" | Guessed wrong table name → run `list_tables` first |
| 4 | "Invalid column name" | Guessed wrong column → run `describe_table` first |
| 5 | Tools invisible despite Connected status | VS Code not restarted → close and reopen |
| 6 | Port 5000 conflict in logs | HTTP mode conflict, harmless in stdio mode → ignore |
| 7 | TLS/certificate error | Missing `encrypt: disable` in source config |

## Files

```
umbraco-mcp-toolbox/
├── SKILL.md       ← skill instructions (read by Claude)
├── README.md      ← this file
└── evals/
    └── evals.json ← benchmark test cases
```
