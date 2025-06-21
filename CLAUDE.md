# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

This is a Python project using `uv` for dependency management and packaging:

- **Run the proxy**: `uv run mcp-proxy [command_or_url] [options]`
- **Run as module**: `uv run -m mcp_proxy`
- **Install for development**: `uv sync`
- **Run tests**: `uv run pytest`
- **Run with coverage**: `uv run coverage run -m pytest && uv run coverage report`
- **Type checking**: `uv run mypy src/`
- **Linting**: Uses ruff (configured in pyproject.toml)

## Architecture Overview

The mcp-proxy is a bidirectional MCP (Model Context Protocol) transport converter with two primary modes:

### Core Components

1. **Entry Point** (`__main__.py`): Argument parsing and mode selection
2. **Proxy Server** (`proxy_server.py`): Creates MCP server instances that proxy requests to remote MCP sessions
3. **MCP Server** (`mcp_server.py`): SSE/HTTP server that manages multiple stdio MCP servers
4. **SSE Client** (`sse_client.py`): Client for connecting to remote SSE MCP servers
5. **StreamableHTTP Client** (`streamablehttp_client.py`): Client for StreamableHTTP transport
6. **Config Loader** (`config_loader.py`): Loads named server configurations from JSON files

### Two Operating Modes

**Mode 1: stdio → SSE/StreamableHTTP (Client Mode)**
- Input: Remote SSE/StreamableHTTP MCP server URL
- Creates a stdio MCP server that proxies to the remote server
- Used by MCP clients (like Claude Desktop) to connect to remote servers

**Mode 2: SSE ← stdio (Server Mode)**  
- Input: Local stdio MCP server command(s)
- Spawns stdio MCP server processes and exposes them via SSE/HTTP endpoints
- Supports multiple named servers with different URL paths
- URLs: `/sse` (default server), `/servers/{name}/sse` (named servers)

### Key Architecture Patterns

- **Async Context Management**: Uses `AsyncExitStack` to manage multiple server lifecycles
- **Proxy Pattern**: `create_proxy_server()` creates MCP servers that forward all requests to remote sessions
- **Transport Abstraction**: Supports both SSE and StreamableHTTP transports
- **Multi-tenancy**: Single proxy instance can manage multiple named MCP servers
- **Configuration**: CLI arguments + JSON config files for named servers

### Server Configuration

Named servers can be defined via:
- CLI: `--named-server name 'command args'`
- JSON: `--named-server-config file.json` (follows MCP client configuration format)

### Important Files

- `src/mcp_proxy/`: Main source code
- `tests/`: Test files using pytest
- `pyproject.toml`: Project configuration, dependencies, and tool settings
- `config_example.json`: Example configuration for named servers