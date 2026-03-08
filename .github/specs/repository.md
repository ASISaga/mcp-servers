# aos-mcp-servers Repository Specification

**Version**: 1.0.0  
**Status**: Active  
**Last Updated**: 2026-03-07

## Overview

`aos-mcp-servers` is a config-driven MCP (Model Context Protocol) server deployment Azure Function app for the Agent Operating System. MCP servers are defined in a JSON registry file and automatically instantiated, managed, and registered with the Foundry Agent Service at runtime. The application exposes REST endpoints for server listing, server lookup, and tool invocation.

## Scope

- Repository role in the AOS ecosystem
- Technology stack and coding patterns
- MCP server registry schema and transport types
- HTTP endpoint design
- Testing and validation workflows
- Key design principles for agents and contributors

## Repository Role

| Concern | Owner |
|---------|-------|
| MCP server registry, transport lifecycle, tool routing | **aos-mcp-servers** |
| Agent declarations of MCP server usage (`MCPServerConfig`) | AOS client applications |
| Foundry Agent Service tool-connection management | AOS / Azure AI Foundry |
| Agent lifecycle, orchestration, messaging | `aos-kernel` |

`aos-mcp-servers` **knows nothing about agent business logic**. It manages transport connections and exposes tools; agents declare which servers they need via `MCPServerConfig`.

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Python 3.10+ |
| App framework | `azure-functions` — raw `func.FunctionApp()` |
| Schema validation | `pydantic` (via `aos-kernel[azure]`) |
| Hosting | Azure Functions |
| Tests | `pytest` + `pytest-asyncio` |
| Linter | `pylint` |
| Build / deploy | `azure.yaml` (Azure Developer CLI) |

## Directory Structure

```
aos-mcp-servers/
├── src/
│   ├── aos_mcp_servers/
│   │   ├── __init__.py
│   │   └── routing.py         # MCPStdioTool, MCPStreamableHTTPTool, MCPWebsocketTool
│   ├── function_app.py        # Azure Functions entry point & HTTP endpoints
│   ├── mcp_server_schema.py   # Pydantic registry schema (MCPServerRegistry, MCPServerRegistryEntry)
│   └── example_mcp_server_registry.json  # Example registry configuration
├── mcp-servers/               # Git submodules for bundled MCP servers
├── tests/
│   ├── test_mcp_routing.py    # Transport class unit tests
│   └── test_mcp_servers.py    # Registry schema & endpoint tests
├── docs/                      # Architecture and API documentation
├── function_app.py            # Thin re-export (entry point for Azure Functions host)
├── pyproject.toml             # Build config, dependencies, pytest settings
└── azure.yaml                 # Azure Developer CLI deployment config
```

## Core Patterns

### Registry-Driven Instantiation

MCP servers are defined in `example_mcp_server_registry.json` (or a path set by the `MCP_REGISTRY_PATH` environment variable) and loaded at first request:

```json
{
    "mcp_servers": [
        {
            "server_id": "context-server",
            "server_type": "stdio",
            "command": "python",
            "args": ["-m", "context_mcp_server"],
            "tools": [{"name": "get_context", "description": "Retrieve agent context"}],
            "enabled": true
        }
    ]
}
```

Each entry's `server_type` determines the transport class instantiated:

| `server_type` | Transport class | Connection |
|---------------|----------------|------------|
| `stdio` | `MCPStdioTool` | Local subprocess via stdin/stdout |
| `streamable_http` | `MCPStreamableHTTPTool` | Remote HTTP + SSE, optional AI Gateway |
| `websocket` | `MCPWebsocketTool` | Persistent WebSocket (`wss://`) |

### Transport Usage

```python
from aos_mcp_servers.routing import (
    MCPStdioTool, MCPStreamableHTTPTool, MCPWebsocketTool,
    MCPToolDefinition, MCPTransportType,
)

# Local process server
stdio_server = MCPStdioTool(
    command="python",
    args=["path/to/server.py"],
    tools=[MCPToolDefinition(name="read_file", description="Read a local file")],
)

# Remote HTTP server with AI Gateway governance
http_server = MCPStreamableHTTPTool(
    url="https://api.example.com/mcp",
    gateway_url="https://my-foundry-gateway.azure.com",
    tools=[MCPToolDefinition(name="search_web", description="Search the internet")],
)
```

### HTTP Endpoints

| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/api/mcp/servers` | List all enabled MCP servers (optional `?server_type=` filter) |
| `GET` | `/api/mcp/servers/{server_id}` | Get a single server descriptor |
| `POST` | `/api/mcp/servers/{server_id}/tools/{tool_name}` | Invoke a tool on a server |
| `GET` | `/api/health` | Health check (returns registry stats) |

### Foundry Agent Registration

On first request, all enabled servers are registered with the Foundry Agent Service via `FoundryAgentManager.register_agent`. The `FOUNDRY_PROJECT_ENDPOINT` environment variable configures the target endpoint.

## Testing Workflow

```bash
# Install dev dependencies
pip install -e ".[dev]"

# Run all tests
pytest tests/ -v

# Lint
pylint src/

# Specific test file
pytest tests/test_mcp_routing.py -v
pytest tests/test_mcp_servers.py -v
```

**CI**: GitHub Actions runs `pytest` across Python 3.10, 3.11, and 3.12 on every push/PR to `main`.

→ **CI workflow**: `.github/workflows/ci.yml`

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_REGISTRY_PATH` | `src/example_mcp_server_registry.json` | Path to the MCP server registry JSON file |
| `FOUNDRY_PROJECT_ENDPOINT` | *(empty)* | Azure AI Foundry project endpoint for tool registration |

## Related Repositories

| Repository | Role |
|-----------|------|
| [aos-kernel](https://github.com/ASISaga/aos-kernel) | OS kernel & Pydantic schemas |
| [aos-dispatcher](https://github.com/ASISaga/aos-dispatcher) | AOS Orchestration API |
| [aos-realm-of-agents](https://github.com/ASISaga/aos-realm-of-agents) | Agent catalog |

## Key Design Principles

1. **Config-driven** — All server definitions live in JSON; no code changes needed to add servers
2. **Transport-agnostic** — One registry schema covers stdio, HTTP, and WebSocket transports
3. **Foundry-integrated** — Servers auto-register with the Foundry Agent Service for tool discovery
4. **Lazy initialization** — Registry and server instances are built on first HTTP request (warm start)

## References

→ **MCP Transport Capabilities Specification**: `.github/specs/mcp-transport-capabilities.md`  
→ **MCP Server Registry & Endpoint Workflows**: `.github/specs/workflows.md`  
→ **Agent framework**: `.github/specs/agent-intelligence-framework.md`  
→ **Conventional tools**: `.github/docs/conventional-tools.md`  
→ **Python coding standards**: `.github/instructions/python.instructions.md`  
→ **MCPServers instructions**: `.github/instructions/mcp-servers.instructions.md`
