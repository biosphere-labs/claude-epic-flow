---
description: Refine rough ideas into technical designs through structured Socratic questioning. Use before epic-create.
allowed-tools: Bash, Read, Write, LS, Task, WebSearch, WebFetch, mcp__brave-search__brave_web_search, mcp__episodic-memory__search, AskUserQuestion
---

# PRD New

Refine a rough idea into a technical design through structured questioning and research.

## Usage
```
/pm:prd-new <feature_name>
```

## Required Rules

**IMPORTANT:** Before executing, read:
- `.claude/rules/datetime.md` - For timestamps

## Phase 0: Prime Context (ALWAYS RUN FIRST)

Before any research or user interaction, load project context:

1. **Check context exists:**
   ```bash
   ls -la workflow/context/*.md 2>/dev/null | wc -l
   ```
   - If 0 files: Tell user "❌ No context found. Run /ccpmcontext:create first." and stop.

2. **Load context files in order:**
   - Read `workflow/context/project-overview.md` - Project understanding
   - Read `workflow/context/tech-context.md` - Technical stack
   - Read `workflow/context/progress.md` - Current status
   - Read `workflow/context/system-patterns.md` - Architecture patterns
   - Read `workflow/context/project-structure.md` - File organization

3. **Brief acknowledgment (don't be verbose):**
   ```
   🧠 Context loaded ({n} files). Project: {name}, Branch: {branch}
   ```

## Preflight

1. **Validate feature name:** kebab-case only (lowercase, numbers, hyphens)
2. **Check existing:** If `workflow/prds/$ARGUMENTS.md` exists, ask to overwrite
3. **Create directory:** `mkdir -p workflow/prds`

## Research Protocol

Before Phase 1, conduct autonomous research:
- Search episodic memory for related past work
- Use WebSearch/brave-search to research similar solutions and patterns
- Read relevant codebase files (existing implementations, data models)
- **Use Context7 MCP (if available)** to fetch official documentation:
  - React patterns and hooks for frontend features
  - NestJS best practices for backend services
  - Library-specific documentation for dependencies
- Form draft model: problem, existing artifacts, external insights, open questions

**Framework documentation research:**
```yaml
Task:
  subagent_type: "general-purpose"
  prompt: |
    Research framework best practices for: $ARGUMENTS

    Tech stack:
    - Frontend: React 18, Vite, TypeScript
    - Backend: NestJS, TypeScript
    - Database: DynamoDB (single-table design)

    Use Context7 MCP if available to fetch:
    - Official documentation for relevant patterns
    - Version-specific considerations
    - Common pitfalls and anti-patterns

    Return: Key insights that should influence the design
```

## Process

```
Progress:
- [ ] Prep: Autonomous recon + web research → draft model
- [ ] Phase 1: Understanding (purpose, constraints, success criteria)
- [ ] Phase 2: Exploration (2-3 approaches, recommend one)
- [ ] Phase 3: Design presentation (architecture, data flow, components)
- [ ] Phase 4: Documentation (workflow/prds/$ARGUMENTS.md)
```

### Prep: Autonomous Recon + Research

- Read repo, docs, git history before asking anything
- Research externally for similar solutions, industry patterns
- Form draft model of the problem and potential solutions
- Open with: "Based on [sources], here's my understanding: [model]. Did I miss anything?"

### Phase 1: Understanding

- Share synthesized understanding first, then invite corrections
- One question at a time, only for gaps you can't close yourself
- Use `AskUserQuestion` for real decision points
- Gather: purpose, constraints, success criteria

### Phase 2: Exploration

- Propose 2-3 approaches
- Lead with your recommendation and why
- Use `AskUserQuestion` for approach selection

**Good output:**
```
Two options for [feature]:

Option 1 (Recommended): [Name]
- [What it does]
- [Key tradeoff]

Option 2: [Name]
- [What it does]
- [Why less preferred]

Recommendation: Option 1 because [reason].

Your call: Approve Option 1?
```

### Phase 3: Design Presentation

- Present design in digestible sections
- Cover: architecture, components, data flow, error handling
- Check in at natural breakpoints: "Stop me if this diverges from what you expect."

### Phase 4: Documentation

Write to `workflow/prds/$ARGUMENTS.md`:

```markdown
---
name: $ARGUMENTS
description: [One-line summary]
status: backlog
created: [ISO datetime from: date -u +"%Y-%m-%dT%H:%M:%SZ"]
github:
---

# Design: $ARGUMENTS

## Summary

[2-3 sentence overview of what we're building and why]

## Problem Statement

[What problem are we solving? Why now?]

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| [decision] | [choice] | [why] |

## Architecture

[Diagram or description of how components interact]

## Data Models

[Schema changes, new entities, storage approach]

## API/Interface Changes

[New endpoints, UI components, integration points]

## Implementation Approach

[High-level steps, what to build first]

## Success Criteria

- [ ] [Measurable criterion]
- [ ] [Measurable criterion]

## Out of Scope

- [What we're explicitly NOT building]

## Dependencies

- [What this depends on]

## Risks

| Risk | Mitigation |
|------|------------|
| [risk] | [how to handle] |
```

## When to Go Backward

- New constraint revealed → Phase 1
- User questions approach → Phase 2
- Requirements unclear → Phase 1

Don't force forward when backward gives better results.

### Phase 5: Sync to GitHub (Backlog)

After creating the PRD file, sync to GitHub as a Backlog issue:

**Project Configuration (from .claude/project.yaml):**
```bash
# Check for project config
if [ ! -f ".claude/project.yaml" ]; then
  echo "⚠️ No project config - skipping GitHub sync"
  # Skip sync but continue with PRD creation
else
  # Extract GitHub repo (owner/repo format)
  GITHUB_REPO=$(grep -A2 "^github:" .claude/project.yaml | grep "repo:" | sed 's/.*repo: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')

  # Optional: Project board settings (if configured)
  PROJECT_NUMBER=$(grep "project_number:" .claude/project.yaml | sed 's/.*project_number: *//' | tr -d ' "')
  PROJECT_OWNER=$(grep "project_owner:" .claude/project.yaml | sed 's/.*project_owner: *//' | tr -d ' "')
  PROJECT_ID=$(grep "project_id:" .claude/project.yaml | sed 's/.*project_id: *//' | tr -d ' "')
  STATUS_FIELD_ID=$(grep "status_field_id:" .claude/project.yaml | sed 's/.*status_field_id: *//' | tr -d ' "')
  STATUS_BACKLOG=$(grep "Backlog_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
fi
```

**1. Check Remote Repository:**
```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"automazeio/ccpm"* ]]; then
  echo "⚠️ Skipping GitHub sync (template repo)"
  # Skip sync but continue
elif [ -z "$GITHUB_REPO" ]; then
  echo "⚠️ Skipping GitHub sync (no GitHub repo configured)"
else
  REPO="$GITHUB_REPO"
fi
```

**2. Create Issue Body:**
```bash
# Extract summary and problem statement for issue body
sed '1,/^---$/d; 1,/^---$/d' workflow/prds/$ARGUMENTS.md > /tmp/prd-body.md

# Add footer
echo "" >> /tmp/prd-body.md
echo "---" >> /tmp/prd-body.md
echo "*This is a PRD (Product Requirements Document). When ready to implement, run \`/pm:epic-create $ARGUMENTS\`*" >> /tmp/prd-body.md
```

**3. Create GitHub Issue:**
```bash
issue_number=$(gh issue create \
  --repo "$REPO" \
  --title "PRD: $ARGUMENTS" \
  --body-file /tmp/prd-body.md \
  --label "prd,backlog" \
  --json number -q .number)

echo "Created issue #${issue_number}"
```

**4. Add to Project (Backlog column):**
```bash
repo=$(gh repo view --json nameWithOwner -q .nameWithOwner)
prd_url="https://github.com/$repo/issues/$issue_number"

# Add to project
item_id=$(gh project item-add $PROJECT_NUMBER \
  --owner "$PROJECT_OWNER" \
  --url "$prd_url" \
  --format json 2>/dev/null | jq -r '.id' 2>/dev/null)

if [ -n "$item_id" ] && [ "$item_id" != "null" ]; then
  # Set to Backlog column
  gh project item-edit \
    --project-id "$PROJECT_ID" \
    --id "$item_id" \
    --field-id "$STATUS_FIELD_ID" \
    --single-select-option-id "$STATUS_BACKLOG" 2>/dev/null
  echo "✅ Added to project → Backlog"
fi
```

**5. Update PRD with GitHub URL:**
```bash
sed -i.bak "/^github:/c\github: $prd_url" workflow/prds/$ARGUMENTS.md
rm workflow/prds/$ARGUMENTS.md.bak 2>/dev/null
```

## Post-Creation

```
═══════════════════════════════════════════════════════════════
  ✅ PRD COMPLETE
═══════════════════════════════════════════════════════════════

  Created: workflow/prds/$ARGUMENTS.md
  GitHub:  #${issue_number} → Backlog
  URL:     ${prd_url}

  📍 Workflow position: PRD created, ready for implementation

  ➡️  Next steps:
     /pm:epic-create $ARGUMENTS     ← Create epic (recommended)
     /pm:prd-sync $ARGUMENTS        ← Re-sync to GitHub if edited

  🔄 Related:
     /pm:prd-status                 ← View all PRDs
     /pm:status                     ← Overall workflow status

═══════════════════════════════════════════════════════════════
```

## $ARGUMENTS

Feature to design: **$ARGUMENTS**
