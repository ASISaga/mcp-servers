# MCP Transport Capabilities Specification

**Version**: 1.0.0
**Status**: Active
**Last Updated**: 2026-03-07

## Overview

`aos-mcp-servers` supports three MCP transport types — `stdio`, `streamable_http`, and `websocket` — each implemented as a dedicated transport class in `aos_mcp_servers.routing`. Every transport class implements a common interface (`list_tools`, `call_tool`) used by the function app to route tool invocations.

## Scope

- Three transport types and their configuration fields
- `MCPToolDefinition` metadata contract
- AI Gateway governance for HTTP transports
- `MCPTransportType` enum values
- Foundry Agent Service registration integration

## Transport Interface

All transport classes expose:

| Method | Signature | Description |
|--------|-----------|-------------|
| `list_tools()` | `async () -> List[MCPToolDefinition]` | Returns tool metadata for this server |
| `call_tool()` | `async (tool_name: str, params: Dict[str, Any]) -> Any` | Invokes a tool; raises `ValueError` for unknown tools |

**Unknown tool handling**: If the server has a non-empty tool list and `tool_name` is not in it, `call_tool` raises `ValueError` with a descriptive message. The HTTP layer converts this to a `404` response.

## MCPToolDefinition

```python
@dataclass
class MCPToolDefinition:
    name: str               # Unique routing key
    description: str = ""  # Human-readable description
    input_schema: Dict[str, Any] = field(default_factory=dict)  # JSON Schema
```

## MCPTransportType Enum

```python
class MCPTransportType(str, Enum):
    STDIO           = "stdio"
    STREAMABLE_HTTP = "streamable_http"
    WEBSOCKET       = "websocket"
```

Values align with the registry `server_type` field and the Microsoft Agent Framework transport names.

## stdio Transport — `MCPStdioTool`

Connects to a local subprocess via stdin/stdout (JSON-RPC over stdio).

**Registry fields**:

| Field | Required | Description |
|-------|----------|-------------|
| `command` | ✓ | Executable to run (e.g. `"python"`) |
| `args` | — | Command-line arguments |
| `env` | — | Additional environment variables |
| `tools` | — | Tool definitions (populated at construction) |

**Construction example**:

```python
MCPStdioTool(
    command="python",
    args=["-m", "context_mcp_server"],
    tools=[MCPToolDefinition(name="get_context", description="Retrieve agent context")],
)
```

**`call_tool` response shape**:

```python
{
    "transport": MCPTransportType.STDIO,
    "tool": tool_name,
    "params": params,
    "result": None,   # populated by actual subprocess in production
}
```

## streamable_http Transport — `MCPStreamableHTTPTool`

Connects to a remote HTTP server using Server-Sent Events (SSE). Supports optional AI Gateway governance.

**Registry fields**:

| Field | Required | Description |
|-------|----------|-------------|
| `url` | ✓ | Base URL of the MCP HTTP endpoint |
| `headers` | — | HTTP headers sent with every request |
| `gateway_url` | — | AI Gateway URL for governed routing |
| `tools` | — | Tool definitions |

**AI Gateway routing**: when `gateway_url` is set, `effective_url` returns it instead of `url`. All requests are routed through the gateway for centralised authentication, rate limiting, and audit logging.

```python
@property
def effective_url(self) -> str:
    return self.gateway_url or self.url
```

**Construction example**:

```python
MCPStreamableHTTPTool(
    url="https://aos-knowledge.azurewebsites.net/mcp",
    gateway_url="https://my-foundry-gateway.azure.com",
    tools=[MCPToolDefinition(name="search_knowledge", description="Search AOS knowledge base")],
)
```

**`call_tool` response shape**:

```python
{
    "transport": MCPTransportType.STREAMABLE_HTTP,
    "tool": tool_name,
    "params": params,
    "url": self.effective_url,
    "result": None,
}
```

## websocket Transport — `MCPWebsocketTool`

Connects to a remote server via a persistent WebSocket for real-time or low-latency bidirectional communication.

**Registry fields**:

| Field | Required | Description |
|-------|----------|-------------|
| `url` | ✓ | WebSocket endpoint (`wss://` or `ws://`) |
| `tools` | — | Tool definitions |

**Construction example**:

```python
MCPWebsocketTool(
    url="wss://aos-realtime.azurewebsites.net/mcp",
    tools=[MCPToolDefinition(name="subscribe_events", description="Subscribe to real-time agent events")],
)
```

**`call_tool` response shape**:

```python
{
    "transport": MCPTransportType.WEBSOCKET,
    "tool": tool_name,
    "params": params,
    "url": self.url,
    "result": None,
}
```

## Foundry Agent Service Registration

All enabled servers are registered with the Foundry Agent Service during `_ensure_foundry_registration()`. Registration is idempotent (runs once per function app process lifetime):

```python
await agent_manager.register_agent(
    agent_id=entry.server_id,
    purpose=entry.description or f"MCP server: {entry.server_id}",
    name=entry.server_id,
    tools=[
        {"type": "function", "function": {"name": t.name, "description": t.description}}
        for t in entry.tools
    ],
)
```

When `FoundryAgentManager` is unavailable (import fails), the application runs in **stub mode** — servers are still initialized and callable, but Foundry registration is skipped.

## Adding a New Transport Type

1. Add the new value to `MCPTransportType` enum
2. Create a new transport class implementing `list_tools()` and `call_tool()`
3. Add a new `if server_type == MCPTransportType.<NEW>:` branch in `_build_server_instance()`
4. Add the new `server_type` value to the registry JSON schema documentation

## Validation

```bash
# Run transport routing tests
pytest tests/test_mcp_routing.py -v

# Run all tests
pytest tests/ -v

# Lint
pylint src/
```

## References

→ **Repository spec**: `.github/specs/repository.md`
→ **MCP Server Registry & Endpoint Workflows**: `.github/specs/workflows.md`
→ **Python standards**: `.github/instructions/python.instructions.md`
→ **MCPServers instructions**: `.github/instructions/mcp-servers.instructions.md`
