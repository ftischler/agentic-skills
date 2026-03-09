# agentic-skills

A personal collection of agentic skills and MCP server configuration for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [GitHub Copilot](https://docs.github.com/en/copilot).

## Repository Structure

```
agentic-skills/
├── claude-code/
│   └── commands/       # Claude Code slash commands (.md files)
├── copilot/
│   └── agents/         # Copilot custom agent definitions (.md files)
└── mcp/
    ├── config.json     # MCP server configuration
    └── README.md       # Per-server setup notes
```

## Usage

### Claude Code

Copy (or symlink) the files from `claude-code/commands/` into your project's `.claude/commands/` directory, or into `~/.claude/commands/` to make them available globally.

```bash
# Copy all commands into a project
cp -r claude-code/commands/ /path/to/project/.claude/commands/

# Or symlink a single command
ln -s $(pwd)/claude-code/commands/my-command.md /path/to/project/.claude/commands/my-command.md
```

### GitHub Copilot

Copy the files from `copilot/agents/` into your project's `.github/agents/` directory.

```bash
cp -r copilot/agents/ /path/to/project/.github/agents/
```

### MCP Configuration

See [`mcp/README.md`](mcp/README.md) for per-server setup instructions.  
Copy `mcp/config.json` as a starting point and adjust to your environment:

```bash
# Claude Code reads MCP config from the project root or ~/.claude/
cp mcp/config.json /path/to/project/.claude/mcp.json
```
