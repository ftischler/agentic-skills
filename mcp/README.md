# MCP Configuration

This directory contains [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server configuration.

## Files

| File          | Purpose |
|---------------|---------|
| `config.json` | MCP server definitions — copy this as a starting point |

## Usage with Claude Code

Claude Code loads MCP configuration from `mcp.json` in the project root (inside `.claude/`) or from `~/.claude/mcp.json` for global servers.

```bash
# Use as a project-level config
cp mcp/config.json /path/to/project/.claude/mcp.json

# Or as a global config
cp mcp/config.json ~/.claude/mcp.json
```

You can also run `claude mcp add` to register servers interactively.

## `config.json` Schema

```jsonc
{
  "mcpServers": {
    "<server-name>": {
      // Launch a local process (stdio transport)
      "command": "npx",
      "args": ["-y", "<package-name>"],
      "env": {
        "API_KEY": "<your-key>"
      }
    },
    "<another-server>": {
      // Connect to a remote server (HTTP/SSE transport)
      "url": "https://example.com/mcp"
    }
  }
}
```

## Adding a New Server

1. Find the server's package name or URL in the [MCP server registry](https://github.com/modelcontextprotocol/servers).
2. Add an entry to `config.json` under `mcpServers`.
3. Store any secrets in environment variables — never hard-code credentials.
