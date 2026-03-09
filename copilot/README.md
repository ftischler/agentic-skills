# Copilot Skills

This directory contains reusable skills for [GitHub Copilot](https://docs.github.com/en/copilot/customizing-copilot/reusing-prompts-and-instructions-in-github-copilot) — specifically custom agent definitions for Copilot coding agents.

## Structure

```
copilot/
└── agents/     # Custom agent definitions — one .md file per agent
```

## Custom Agents

Custom agents are Markdown files with YAML front matter, placed in `.github/agents/`.  
Each file defines a named agent with a description and a set of instructions.

### File Format

```markdown
---
name: Agent Name
description: A short description shown in the agent picker
tools:
  - read_file
  - create_file
  - run_command
---

Detailed instructions that tell the agent how to behave.
Reference any relevant standards, patterns, or constraints here.
```

#### Front Matter Fields

| Field         | Required | Description |
|---------------|----------|-------------|
| `name`        | Yes      | Display name of the agent |
| `description` | Yes      | One-line summary shown in the agent picker |
| `tools`       | No       | List of tools the agent is allowed to use |
| `model`       | No       | Override the default model (e.g. `gpt-4o`) |

### Installing Agents

```bash
# Copy all agents into a project
cp -r copilot/agents/ /path/to/project/.github/agents/
```
