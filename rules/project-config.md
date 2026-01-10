# Project Configuration

All PM commands that interact with GitHub or need project-specific settings MUST read from `.claude/project.yaml`.

## Reading Project Config

Before any GitHub operation, read the project configuration:

```bash
# Check for project config
if [ ! -f ".claude/project.yaml" ]; then
  echo "❌ No project configuration found"
  echo "   Run: /pm:init to configure this project"
  exit 1
fi

# Extract GitHub repo (owner/repo format)
GITHUB_REPO=$(grep -A1 "^github:" .claude/project.yaml | grep "repo:" | sed 's/.*repo: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')

# Extract default branch
DEFAULT_BRANCH=$(grep -A2 "^github:" .claude/project.yaml | grep "default_branch:" | sed 's/.*default_branch: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')

# Extract project name
PROJECT_NAME=$(grep -A1 "^project:" .claude/project.yaml | grep "name:" | sed 's/.*name: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')

# Validate required fields
if [ -z "$GITHUB_REPO" ]; then
  echo "❌ GitHub repo not configured in .claude/project.yaml"
  exit 1
fi
```

## Using Project Config

### GitHub Operations

Always use `--repo` flag with the configured repo:

```bash
# Creating issues
gh issue create --repo "$GITHUB_REPO" --title "..." --body-file ...

# Viewing issues
gh issue view --repo "$GITHUB_REPO" 123

# Editing issues
gh issue edit --repo "$GITHUB_REPO" 123 --body-file ...
```

### Project-Specific Settings

Access optional settings:

```bash
# API keys (if configured)
WEBSITE_API_KEY=$(grep -A1 "api_keys:" .claude/project.yaml | grep "website:" | sed 's/.*website: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')
```

## Config File Location

- **Always** look for `.claude/project.yaml` in the current project root
- **Never** hardcode repository URLs in commands
- **Never** assume a specific project structure

## Error Handling

If project.yaml is missing or invalid:
1. Show clear error message
2. Suggest running `/pm:init`
3. Do NOT fall back to git remote (that's how wrong-repo issues happen)

## Example project.yaml

```yaml
project:
  name: "my-project"
  description: "A cool project"

github:
  repo: "myorg/my-project"
  default_branch: "staging"

api_keys:
  website: "sk-..."
```
