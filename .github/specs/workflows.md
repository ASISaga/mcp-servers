# MCP Server Registry & Endpoint Workflows Specification

**Version**: 1.0.0
**Status**: Active
**Last Updated**: 2026-03-07

## Overview

`aos-mcp-servers` exposes four HTTP endpoints that manage the lifecycle of config-driven MCP servers: list, lookup, tool invocation, and health. All servers are defined in a JSON registry and auto-registered with the Foundry Agent Service on first request.

## Scope

- HTTP endpoint contracts (request, response, error shapes)
- Registry loading and server initialization lifecycle
- Foundry Agent Service registration pattern
- Tool invocation routing

## Endpoint Workflows

### `GET /api/mcp/servers` — List Servers

Returns all enabled MCP servers from the registry.

| Step | Action |
|------|--------|
| 1 | `_load_registry()` — load JSON from `MCP_REGISTRY_PATH` (cached after first call) |
| 2 | `_ensure_servers_initialized(registry)` — build transport instances for each enabled server |
| 3 | `_ensure_foundry_registration(registry)` — register servers with Foundry Agent Service (idempotent) |
| 4 | Filter by optional `?server_type=` query parameter |
| 5 | Return `{"servers": [<MCPServerRegistryEntry.model_dump()>, ...]}` |

**Query parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `server_type` | string | Optional filter: `stdio`, `streamable_http`, or `websocket` |

**Response** (`200 OK`):

```json
{
    "servers": [
        {
            "server_id": "context-server",
            "server_type": "stdio",
            "description": "Provides agent context and state management tools",
            "enabled": true,
            "tools": [{"name": "get_context", "description": "Retrieve the current agent execution context", "input_schema": {}}]
        }
    ]
}
```

---

### `GET /api/mcp/servers/{server_id}` — Get Server

Returns a single server descriptor by ID.

**Response** (`200 OK`): full `MCPServerRegistryEntry` as JSON.

**Error** (`404 Not Found`):

```json
{"error": "MCP server 'unknown-server' not found"}
```

---

### `POST /api/mcp/servers/{server_id}/tools/{tool_name}` — Invoke Tool

Invokes a named tool on the specified MCP server.

**Request body**: JSON object of tool input parameters (may be empty `{}`).

**Response** (`200 OK`): transport-specific result dict.

```json
{
    "transport": "stdio",
    "tool": "get_context",
    "params": {"agent_id": "ceo"},
    "result": null
}
```

**Error** (`404 Not Found`) — server not found or not initialized:

```json
{"error": "MCP server 'unknown-server' not found or not initialized"}
```

**Error** (`404 Not Found`) — tool not found on server:

```json
{"error": "Tool 'unknown_tool' is not available on stdio server (command='python')"}
```

---

### `GET /api/health` — Health Check

Returns registry status. Used by Azure health probes.

**Response** (`200 OK`):

```json
{
    "app": "aos-mcp-servers",
    "status": "healthy",
    "servers_registered": 3,
    "servers_enabled": 2
}
```

**Response** (`503 Service Unavailable`) — registry load failed:

```json
{
    "app": "aos-mcp-servers",
    "status": "unhealthy",
    "error": "<exception message>"
}
```

## Initialization Lifecycle

```
HTTP request arrives
       │
       ▼
_load_registry()               ← reads JSON once, cached in _registry
       │
       ▼
_ensure_servers_initialized()  ← builds MCPStdioTool / MCPStreamableHTTPTool / MCPWebsocketTool
       │                          for each enabled server (skips already-built instances)
       ▼
_ensure_foundry_registration() ← registers tools with FoundryAgentManager (once per process)
```

## Server Type Routing

| `server_type` in registry | Transport class | Required fields |
|---------------------------|----------------|-----------------|
| `stdio` | `MCPStdioTool` | `command` |
| `streamable_http` | `MCPStreamableHTTPTool` | `url` |
| `websocket` | `MCPWebsocketTool` | `url` |

Unsupported `server_type` values raise `ValueError` during initialization and are logged as warnings (the server is skipped).

## Registry Configuration Pattern

```json
{
    "mcp_servers": [
        {
            "server_id": "<unique-id>",
            "server_type": "stdio | streamable_http | websocket",
            "description": "<human-readable>",
            "command": "<executable>",
            "args": ["<arg1>"],
            "env": {"KEY": "value"},
            "url": "<http-or-ws-url>",
            "headers": {"Authorization": "Bearer <token>"},
            "gateway_url": "<ai-gateway-url>",
            "tools": [
                {"name": "<tool-name>", "description": "<desc>", "input_schema": {}}
            ],
            "enabled": true
        }
    ]
}
```

Set `enabled: false` to deactivate a server without removing it from the registry.

## Validation

```bash
# Run all tests
pytest tests/ -v

# Run endpoint/schema tests only
pytest tests/test_mcp_servers.py -v

# Run transport routing tests only
pytest tests/test_mcp_routing.py -v

# Lint
pylint src/
```

## References

→ **Repository spec**: `.github/specs/repository.md`
→ **MCP Transport Capabilities Specification**: `.github/specs/mcp-transport-capabilities.md`
→ **Python standards**: `.github/instructions/python.instructions.md`
→ **MCPServers instructions**: `.github/instructions/mcp-servers.instructions.md`
