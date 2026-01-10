# Worktree Operations

Git worktrees enable parallel development by allowing multiple working directories for the same repository.

## Branch Model

This project uses a **two-branch model**:

```
feature/xyz ──→ staging ──→ main
              (PR)       (PR)
```

- **staging**: Integration branch for testing (deploys to staging environment)
- **main**: Production branch (deploys to production environment)
- **feature/epic branches**: Created from staging, merged back to staging via PR

## Worktree Location

All worktrees are created inside the project's `.worktrees/` directory:
```
.worktrees/
  epic/{name}/        # Epic worktrees
  feat/{name}/        # Feature worktrees
  bugfix/{name}/      # Bugfix worktrees
```

This keeps worktrees organized within the project rather than as siblings.

## Creating Worktrees

Always create worktrees from a clean staging branch:
```bash
# Ensure staging is up to date
git checkout staging
git pull origin staging

# Create worktree for epic
mkdir -p .worktrees/epic
git worktree add .worktrees/epic/{name} -b epic/{name}
```

## Setting Up a Worktree (Monorepo)

After creating a worktree, you must copy gitignored dependencies and config files. This is a monorepo with `backend/`, `frontend/`, and `packages/shared-types/`.

### Required Setup Steps

```bash
# From main repository, after creating worktree
EPIC_NAME={name}
WORKTREE_PATH=.worktrees/epic/$EPIC_NAME

# 1. Copy node_modules (faster than fresh install)
cp -r backend/node_modules $WORKTREE_PATH/backend/
cp -r frontend/node_modules $WORKTREE_PATH/frontend/
cp -r packages/shared-types/node_modules $WORKTREE_PATH/packages/shared-types/

# 2. Copy shared-types dist (built artifacts are gitignored)
cp -r packages/shared-types/dist $WORKTREE_PATH/packages/shared-types/

# 3. Copy .env files (required for app to run)
cp backend/.env $WORKTREE_PATH/backend/.env
cp frontend/.env $WORKTREE_PATH/frontend/.env
```

### Why This Is Needed

- `node_modules/` is gitignored - worktrees don't have dependencies
- `packages/shared-types/dist/` is gitignored - backend/frontend won't compile
- `.env` files are gitignored - app won't start without config

### Quick One-Liner

```bash
EPIC={name} && cp -r backend/node_modules frontend/node_modules packages/shared-types/{node_modules,dist} .worktrees/epic/$EPIC/ 2>/dev/null; cp backend/.env .worktrees/epic/$EPIC/backend/; cp frontend/.env .worktrees/epic/$EPIC/frontend/
```

## Working in Worktrees

### Agent Commits
- Agents commit directly to the worktree
- Use small, focused commits
- Commit message format: `Issue #{number}: {description}`
- Example: `Issue #1234: Add user authentication schema`

### File Operations
```bash
# Working directory is the worktree
cd .worktrees/epic/{name}

# Normal git operations work
git add {files}
git commit -m "Issue #{number}: {change}"

# View worktree status
git status
```

## Parallel Work in Same Worktree

Multiple agents can work in the same worktree if they touch different files:
```bash
# Agent A works on API
git add src/api/*
git commit -m "Issue #1234: Add user endpoints"

# Agent B works on UI (no conflict!)
git add src/ui/*
git commit -m "Issue #1235: Add dashboard component"
```

## Merging Worktrees

When epic is complete, merge back to staging (then staging → main for production):
```bash
# From main repository (not worktree)
cd {main-repo}
git checkout staging
git pull origin staging

# Merge epic branch
git merge epic/{name}

# If successful, clean up
git worktree remove .worktrees/epic/{name}
git branch -d epic/{name}

# Push to staging
git push origin staging

# When ready for production, create PR: staging → main
```

## Handling Conflicts

If merge conflicts occur:
```bash
# Conflicts will be shown
git status

# Human resolves conflicts
# Then continue merge
git add {resolved-files}
git commit
```

## Worktree Management

### List Active Worktrees
```bash
git worktree list
```

### Remove Stale Worktree
```bash
# If worktree directory was deleted
git worktree prune

# Force remove worktree
git worktree remove --force .worktrees/epic/{name}
```

### Check Worktree Status
```bash
# From main repo
cd .worktrees/epic/{name} && git status && cd -
```

## Best Practices

1. **One worktree per epic** - Not per issue
2. **Clean before create** - Always start from updated staging
3. **Commit frequently** - Small commits are easier to merge
4. **Delete after merge** - Don't leave stale worktrees
5. **Use descriptive branches** - `epic/feature-name` not `feature`

## Common Issues

### Worktree Already Exists
```bash
# Remove old worktree first
git worktree remove .worktrees/epic/{name}
# Then create new one
```

### Branch Already Exists
```bash
# Delete old branch
git branch -D epic/{name}
# Or use existing branch
git worktree add .worktrees/epic/{name} epic/{name}
```

### Cannot Remove Worktree
```bash
# Force removal
git worktree remove --force .worktrees/epic/{name}
# Clean up references
git worktree prune
```