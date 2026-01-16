# Merge Strategy Rule

Git merge strategy varies by target branch to balance traceability with clean history.

## Branch Flow

```
feature/bugfix → staging → main
     (merge)      (squash)
```

## Feature/Bugfix → Staging: Regular Merge

**Never squash** when merging to staging:

```bash
# Correct - preserves commit history
gh pr merge --merge --delete-branch

# Wrong - destroys granular history
gh pr merge --squash --delete-branch
```

**Why:**
- Granular commits enable `git bisect` for debugging
- Preserves the development story for code review
- Individual commits show incremental changes
- Better traceability when investigating issues

## Staging → Main: Squash Merge

**Squash** when merging staging to main (production):

```bash
# Correct - clean production history
gh pr merge --squash

# Also acceptable - if you want full history in prod too
gh pr merge --merge
```

**Why:**
- Production history stays clean
- Each release is one atomic commit
- Easier to revert entire releases
- Release notes map cleanly to commits

## Quick Reference

| Source | Target | Strategy | Command |
|--------|--------|----------|---------|
| feature/* | staging | merge | `gh pr merge --merge --delete-branch` |
| bugfix/* | staging | merge | `gh pr merge --merge --delete-branch` |
| epic/* | staging | merge | `gh pr merge --merge --delete-branch` |
| staging | main | squash | `gh pr merge --squash` |

## Delete Branch After Merge

Always delete the source branch after merging to staging:
- Keeps branch list clean
- The commits are preserved in staging
- Can always recreate from commit history if needed

**Exception:** Don't delete `staging` after merging to `main`.

## Reverting

If you need to undo:
- **Feature merge to staging:** `git revert <merge-commit>` or revert individual commits
- **Staging squash to main:** `git revert <squash-commit>` reverts the entire release

## Default Behavior for Ad-Hoc PR Merges

When asked to merge a PR outside of PM commands (e.g., "merge this PR"), **always use regular merge by default**:

```bash
# Default behavior - regular merge
gh pr merge --merge --delete-branch

# Only use squash if explicitly requested OR if target is main
gh pr merge --squash --delete-branch
```

**Ask for clarification** if unsure about the target branch:
- If merging to staging/develop: Use `--merge`
- If merging to main/production: Use `--squash`

## Important Notes

- Never force push to staging or main
- The `--delete-branch` flag is safe - commits are preserved
- Squashing loses individual commit messages, so ensure PR description is comprehensive
