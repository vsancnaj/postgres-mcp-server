# PostgreSQL MCP Server

An MCP server that exposes PostgreSQL database operations as tools for AI assistants.

## What is MCP?

Model Context Protocol (MCP) connects AI assistants to external tools and data. This server lets AI assistants execute SQL queries and inspect your PostgreSQL database schema.

## Setup

**Python version:** This project currently supports Python 3.10–3.13. Python 3.14 is excluded due to upstream dependency wheel support (e.g., `psycopg2-binary` and `pydantic-core`).

### 1. Install Dependencies

> **Note:** This project uses Poetry for dependency management. If you don't have Poetry installed, you can install it with:
> ```bash
> curl -sSL https://install.python-poetry.org | python3 -
> ```
> See the [official Poetry documentation](https://python-poetry.org/docs/#installation) for alternative installation methods.

```bash
poetry install
```

### 2. Configure Database

Copy `.env.example` to `.env` and add your PostgreSQL credentials:

```bash
cp .env.example .env
```

Edit `.env`:
```
DB_NAME=your_database
DB_USER=postgres
DB_PASSWORD=your_password
DB_HOST=localhost
DB_PORT=5432
```

## Testing

### MCP Inspector (Recommended)

> **Note:** The Inspector runs via `npx`, so you need Node.js (which includes npm). If you don't have it installed, get it from the official Node.js installer.

```bash
npx @modelcontextprotocol/inspector poetry run python postgres-mcp-server/main.py
```

This opens a web UI where you can:
- View available tools under the **Tools** tab
- Test `get_schema`
- See real-time results

### Quick Test

```bash
poetry run python postgres-mcp-server/main.py
```

Press `Ctrl+C` to stop. No errors = working correctly.

## Available Tools

**`list_tables()`** - List the tables in the public schema

**`get_schema(table)`** - Get column names and types for one table

**`execute_sql(query)`** - Run any SQL. Queries that return rows give a list of `{column_name: value}`. INSERT, UPDATE and DELETE give `[{"rows_affected": N}]`. It runs whatever it is sent, so use a practice database or a read-only user.

After you add or change a tool in `main.py`, reconnect the server: open `/mcp` in Claude Code, pick `postgres`, then Reconnect. A running server keeps its old tool list.

## Connect to Claude Code

Add a file named `.mcp.json` at the root of the project you want to use it from (Claude Code reads it on start):

```json
{
  "mcpServers": {
    "postgres": {
      "command": "poetry",
      "args": ["-C", "/absolute/path/to/postgres-mcp-server", "run", "python", "/absolute/path/to/postgres-mcp-server/postgres-mcp-server/main.py"]
    }
  }
}
```

Replace both `/absolute/path/to/postgres-mcp-server` values with your actual project path. Use the full path to `main.py` too. A relative path only works when you start `claude` inside this folder.

Or register it from the terminal instead. Run this as one line (see Troubleshooting if it wraps):

```bash
claude mcp add --scope user --transport stdio postgres -- poetry -C /absolute/path/to/postgres-mcp-server run python /absolute/path/to/postgres-mcp-server/postgres-mcp-server/main.py
```

Then start `claude` and ask it to list the tables. The three tools should appear under the server's tools.

## Troubleshooting: "Connection closed" in /mcp

Three setup problems caused this, one after another. Check them in order.

1. **A long command wrapped when pasted.** The terminal splits it, and the saved command ends early (for example at `run python`). Run `claude mcp get postgres` and check that the Args line ends with the full path to `main.py`. If not, run `claude mcp remove postgres --scope user`, then add it again as one line.
2. **Packages are not installed.** Error: `ModuleNotFoundError: No module named 'psycopg2'`. Fix: run `poetry install` in this folder.
3. **Relative path to `main.py`.** Error: `can't open file '.../postgres-mcp-server/main.py'`. `poetry -C` does not change the folder Python looks in. Use the full path to `main.py`.

`claude mcp get` can say Connected while the server is still broken. To really test it, start the server and ask for the tool list:

```bash
cd /tmp && printf '%s\n%s\n%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"0"}}}' '{"jsonrpc":"2.0","method":"notifications/initialized"}' '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' | poetry -C /absolute/path/to/postgres-mcp-server run python /absolute/path/to/postgres-mcp-server/postgres-mcp-server/main.py
```

You should see `list_tables`, `get_schema` and `execute_sql` in the output. Running it from `/tmp` proves the path works from any folder.

---

*Future Proof Data Science - Teaching data scientists to optimize workflows with AI*
