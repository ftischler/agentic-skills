# Claude Code Skills

This directory contains reusable skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Structure

```
claude-code/
└── commands/       # Slash commands — one .md file per command
```

## Slash Commands

Claude Code slash commands are Markdown files placed in `.claude/commands/`.  
Each file becomes a `/command-name` you can invoke inside a Claude Code session.

### File Format

Create a `.md` file whose name matches the command you want (e.g. `review-pr.md` → `/review-pr`):

```markdown
# Review Pull Request

Review the current branch's changes against main, checking for:

- Correctness and logic errors
- Security issues
- Code style consistency
- Missing tests

Provide a summary and an actionable list of suggestions.
```

### Subdirectory Namespacing

Nest commands in subdirectories to namespace them:

```
commands/
└── git/
    ├── commit.md  →  /git:commit
    └── rebase.md  →  /git:rebase
```

### Installing Commands

**Project-level** (only for a specific project):
```bash
cp commands/my-command.md /path/to/project/.claude/commands/my-command.md
```

**Global** (available in every project):
```bash
cp commands/my-command.md ~/.claude/commands/my-command.md
```
