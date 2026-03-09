---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection, safety verification, and pnpm/Nx monorepo initialization. Use this skill whenever the user mentions worktrees, isolated workspaces, parallel branches, or wants to start feature work in a separate environment, even if they don't explicitly say "worktree".
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching. This skill is tailored for our pnpm-based Nx monorepo.

**Core principle:** Systematic directory selection + safety verification + proper initialization = reliable isolation.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Directory Selection Process

Follow this priority order:

### 1. Check Existing Directories

```bash
# Check in priority order
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```

**If found:** Use that directory. If both exist, `.worktrees` wins.

### 2. Check CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**If preference specified:** Use it without asking.

### 3. Ask User

If no directory exists and no CLAUDE.md preference:

```
No worktree directory found. Where should I create worktrees?

1. .worktrees/ (project-local, hidden)
2. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## Safety Verification

### For Project-Local Directories (.worktrees or worktrees)

Verify the directory is gitignored before creating the worktree — otherwise worktree contents get tracked and pollute git status:

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**If NOT ignored:** Fix it immediately:

1. Add the directory to `.gitignore`
2. Commit the change
3. Proceed with worktree creation

### For Global Directory (~/.config/superpowers/worktrees)

No .gitignore verification needed — outside the project entirely.

## Creation Steps

### 1. Detect Project Name

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. Generate a Meaningful Branch Name

The branch name should describe the work being done, not be a generic worktree label. Derive it from the task or feature the user is working on, using the repo's conventional prefix format:

| Prefix      | Use for                   |
| ----------- | ------------------------- |
| `feat/`     | New features              |
| `fix/`      | Bug fixes                 |
| `refactor/` | Code restructuring        |
| `chore/`    | Maintenance, config, deps |

**Example:** If the user says "I need to add API key rotation", the branch name should be something like `feat/api-key-rotation` — not `worktree-1` or `add-worktree-skill`.

If the intent is unclear, ask the user what they're working on so you can pick a good name. The worktree directory name can be short (e.g., `api-key-rotation`), but the branch name should always carry a meaningful prefix.

### 3. Create Worktree

```bash
# Determine full path (use a short directory name derived from the branch)
case $LOCATION in
  .worktrees|worktrees)
    path="$LOCATION/$SHORT_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$SHORT_NAME"
    ;;
esac

# Create worktree with a descriptive branch name
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 4. Initialize Worktree

Every worktree needs two things before it's usable: dependencies and environment config. Without either, builds and dev servers will fail.

**Step 1 — Install dependencies:**

```bash
pnpm install
```

**Step 2 — Symlink .env from the monorepo root:**

The `.env` file lives in the main repo root and contains secrets/config that all apps need. Since worktrees are separate directory trees, they don't share this file automatically — so we symlink it.

```bash
ln -s /Users/ftischler/Development/monorepo/.env .env
```

Both steps are mandatory. Never skip the `.env` symlink — without it, apps will start with missing environment variables and fail in confusing ways.

### 5. Verify Clean Baseline

Run a quick sanity check to make sure the worktree starts clean:

```bash
pnpm nx run-many --target=build --all --dry-run
```

Or run tests for the specific project you'll be working on:

```bash
pnpm nx test <project> --excludeTaskDependencies
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 6. Report Location

```
Worktree ready at <full-path>
Branch: <branch-name>
Dependencies installed, .env linked
Ready to implement <feature-name>
```

## Quick Reference

| Situation                  | Action                     |
| -------------------------- | -------------------------- |
| `.worktrees/` exists       | Use it (verify ignored)    |
| `worktrees/` exists        | Use it (verify ignored)    |
| Both exist                 | Use `.worktrees/`          |
| Neither exists             | Check CLAUDE.md → Ask user |
| Directory not ignored      | Add to .gitignore + commit |
| Tests fail during baseline | Report failures + ask      |
| `.env` missing in worktree | Symlink from monorepo root |

## Common Mistakes

### Skipping ignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktree

### Forgetting the .env symlink

- **Problem:** Apps fail to start because environment variables are missing
- **Fix:** Always run `ln -s /Users/ftischler/Development/monorepo/.env .env` after `pnpm install`

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: existing > CLAUDE.md > ask

### Using generic branch names

- **Problem:** Branches like `worktree-1` or `test-branch` are meaningless in git log and MR lists
- **Fix:** Always generate a descriptive branch name with a conventional prefix (`feat/`, `fix/`, etc.) based on the actual task

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

## Example Workflow

```
User: "I need to add API key rotation to the backend"

You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Generate branch name from task: feat/api-key-rotation]
[Create worktree: git worktree add .worktrees/api-key-rotation -b feat/api-key-rotation]
[Run pnpm install]
[Symlink .env: ln -s /Users/ftischler/Development/monorepo/.env .env]
[Run pnpm nx test nemovote-server --excludeTaskDependencies - passing]

Worktree ready at /Users/ftischler/Development/monorepo/.worktrees/api-key-rotation
Branch: feat/api-key-rotation
Dependencies installed, .env linked
Ready to implement API key rotation
```

## Red Flags

**Never:**

- Create worktree without verifying it's ignored (project-local)
- Use generic or meaningless branch names (`worktree-1`, `test`, `branch-2`)
- Skip `pnpm install`
- Skip the `.env` symlink
- Proceed with failing tests without asking
- Assume directory location when ambiguous

**Always:**

- Generate a descriptive branch name with conventional prefix (`feat/`, `fix/`, `refactor/`, `chore/`)
- Follow directory priority: existing > CLAUDE.md > ask
- Verify directory is ignored for project-local
- Run `pnpm install` then symlink `.env`
- Verify clean test baseline

## Integration

**Called by:**

- Any skill or workflow needing an isolated workspace
- Feature branches that should not interfere with current work

**Pairs with:**

- Worktree cleanup: `git worktree remove <path>` when done
