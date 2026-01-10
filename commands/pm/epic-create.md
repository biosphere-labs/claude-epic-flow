---
allowed-tools: Bash, Read, Write, LS, Task, mcp__episodic-memory__search, AskUserQuestion
---

# Epic Create

Create a technical implementation epic from an idea or PRD. If the PRD has a GitHub issue, this command promotes it from Backlog to ToDo.

## Project Configuration

Read from `.claude/project.yaml`:

```bash
# Check for project config
if [ ! -f ".claude/project.yaml" ]; then
  echo "⚠️ No project config - GitHub sync will be skipped"
  # Continue with epic creation
else
  # Extract GitHub repo (owner/repo format)
  GITHUB_REPO=$(grep -A2 "^github:" .claude/project.yaml | grep "repo:" | sed 's/.*repo: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')

  # Optional: Project board settings (if configured)
  PROJECT_NUMBER=$(grep "project_number:" .claude/project.yaml | sed 's/.*project_number: *//' | tr -d ' "')
  PROJECT_OWNER=$(grep "project_owner:" .claude/project.yaml | sed 's/.*project_owner: *//' | tr -d ' "')
  PROJECT_ID=$(grep "project_id:" .claude/project.yaml | sed 's/.*project_id: *//' | tr -d ' "')
  STATUS_FIELD_ID=$(grep "status_field_id:" .claude/project.yaml | sed 's/.*status_field_id: *//' | tr -d ' "')
  STATUS_TODO=$(grep "ToDo_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
fi
```

## Usage
```
/pm:epic-create <feature_name> [description]
```

## Entry Point Guidance

**Before creating an epic, determine the right entry point:**

### When to use `/pm:prd-new` first:
- Large features requiring stakeholder alignment
- Features with complex business rules or user flows
- Cross-team dependencies or external integrations
- Regulatory or compliance requirements
- Features needing formal approval process

### When to use `/pm:epic-create` directly:
- Clear, well-defined features
- Technical improvements or refactoring
- Features where you're the decision maker
- Quick iterations on existing functionality

### When to use `/pm:bugfix` instead:
- Fixing broken functionality
- Performance issues
- Error corrections

**If the user's input suggests they need a PRD first, guide them:**
```
This looks like a larger feature that might benefit from a PRD first.

A PRD helps when:
- Multiple user flows need to be defined
- Business requirements need documentation
- Stakeholder alignment is needed

Options:
1. /pm:prd-new {feature} - Create PRD first (recommended for complex features)
2. /pm:epic-create {feature} - Proceed directly (for simpler, well-defined work)

Which would you prefer?
```

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

## Instructions

### 1. Check for Existing PRD

```bash
if test -f workflow/prds/$ARGUMENTS.md; then
  echo "Found PRD: workflow/prds/$ARGUMENTS.md"
  # Use PRD as source
else
  echo "No PRD found - will create epic from description"
  # Create epic from user's description/idea
fi
```

### 2. Gather Context

**Search episodic memory** for related past work:
```
mcp__episodic-memory__search: "$ARGUMENTS feature implementation"
```

**Explore codebase** for related patterns:
```yaml
Task:
  subagent_type: "Explore"
  prompt: "Find code related to $ARGUMENTS - existing patterns, data models, API structure"
```

**Research framework best practices** (use Context7 MCP if available):
```yaml
Task:
  subagent_type: "general-purpose"
  prompt: |
    Research best practices for implementing $ARGUMENTS feature.

    Tech stack:
    - Frontend: React, Vite, TypeScript
    - Backend: NestJS, TypeScript
    - Database: DynamoDB

    Use Context7 MCP if available to fetch:
    - Official React patterns for this type of feature
    - NestJS best practices for the backend approach
    - Any relevant library documentation

    Return: Key patterns, approaches, and any version-specific considerations
```

### 3. Spec Flow Analysis

Analyze user flows and identify gaps:

```yaml
Task:
  subagent_type: "general-purpose"
  prompt: |
    Analyze this feature for completeness:

    Feature: $ARGUMENTS
    Description: {from PRD or user input}

    Identify:
    1. All user flows (primary and alternative)
    2. Edge cases and error states
    3. Missing requirements or ambiguities
    4. Security considerations
    5. Performance implications

    Return a structured analysis.
```

### 4. Create Epic File

Create at `workflow/epics/$ARGUMENTS/epic.md`:

```markdown
---
name: $ARGUMENTS
status: backlog
created: {ISO datetime from: date -u +"%Y-%m-%dT%H:%M:%SZ"}
progress: 0%
prd: {workflow/prds/$ARGUMENTS.md if exists, otherwise "none"}
github:
---

# Epic: $ARGUMENTS

## Overview
{Brief technical summary - 2-3 sentences}

## User Flows

### Primary Flow
1. {step}
2. {step}
...

### Alternative Flows
- {flow}: {description}

### Edge Cases
- {case}: {handling}

## Technical Approach

### Frontend
- {component/change needed}

### Backend
- {service/endpoint needed}

### Data Model
- {schema changes if any}

## Architecture Decisions
| Decision | Choice | Rationale |
|----------|--------|-----------|
| {decision} | {choice} | {why} |

## Task Breakdown Preview
- [ ] {Task 1 - keep high level}
- [ ] {Task 2}
- [ ] {Task 3}
...

**Target: ≤10 tasks total**

## Dependencies
- {dependency}

## Risks
| Risk | Mitigation |
|------|------------|
| {risk} | {mitigation} |

## Success Criteria
- [ ] {criterion}

## Reviewed Decisions

Decisions reviewed during simplicity analysis and intentionally kept:

| Decision | Concern Raised | Rationale for Keeping |
|----------|----------------|----------------------|
| (none yet) | - | - |
```

### 5. Promote PRD Issue to ToDo (if exists)

If the PRD had a GitHub issue, promote it:

```bash
# Check if PRD has a GitHub issue
PRD_FILE="workflow/prds/$ARGUMENTS.md"
if [ -f "$PRD_FILE" ]; then
  prd_github=$(grep '^github:' "$PRD_FILE" | sed 's/^github: *//')

  if [ -n "$prd_github" ] && [ "$prd_github" != "" ]; then
    issue_num=$(echo "$prd_github" | grep -oE '[0-9]+$')

    # Update issue title from "PRD: xxx" to "Epic: xxx"
    gh issue edit "$issue_num" \
      --title "Epic: $ARGUMENTS" \
      --remove-label "prd,backlog" \
      --add-label "epic,feature" 2>/dev/null

    # Move to ToDo column in project (if project board configured)
    if [ -n "$PROJECT_NUMBER" ] && [ -n "$PROJECT_OWNER" ]; then
      item_id=$(gh project item-list $PROJECT_NUMBER \
        --owner "$PROJECT_OWNER" \
        --format json 2>/dev/null | \
        jq -r --arg url "$prd_github" '.items[] | select(.content.url == $url) | .id' 2>/dev/null)

      if [ -n "$item_id" ] && [ "$item_id" != "null" ] && [ -n "$PROJECT_ID" ] && [ -n "$STATUS_FIELD_ID" ] && [ -n "$STATUS_TODO" ]; then
        gh project item-edit \
          --project-id "$PROJECT_ID" \
          --id "$item_id" \
          --field-id "$STATUS_FIELD_ID" \
          --single-select-option-id "$STATUS_TODO" 2>/dev/null
        echo "✅ Promoted PRD issue #$issue_num → ToDo"
      fi
    else
      echo "✅ Promoted PRD issue #$issue_num (no project board configured)"
    fi

    # Update epic file with the same GitHub URL
    sed -i.bak "/^github:/c\github: $prd_github" workflow/epics/$ARGUMENTS/epic.md
    rm workflow/epics/$ARGUMENTS/epic.md.bak 2>/dev/null
  fi
fi
```

### 6. Simplicity Review & Iterative Refinement

After creating the epic file, analyze it for over-engineering and YAGNI violations:

```yaml
Task:
  subagent_type: "general-purpose"
  prompt: |
    You are a code simplicity expert applying YAGNI principles. Analyze this epic:

    {read workflow/epics/$ARGUMENTS/epic.md}

    First, check the "## Reviewed Decisions" section for already-reviewed items.
    Skip any concerns that match decisions already listed there.

    For NEW concerns only, identify:
    - Unnecessary abstractions or premature generalizations
    - Features not explicitly required by the core use case
    - Overly complex approaches when simpler alternatives exist
    - Architecture decisions that add complexity without clear benefit

    For each concern, provide:
    - What: The specific issue (quote the relevant section)
    - Why: Why it violates YAGNI or adds unnecessary complexity
    - Simpler alternative: What to do instead

    Return a structured list of concerns (max 5 most impactful).
    If the epic is already minimal and focused, say "No simplicity concerns found."
```

**Iterative Refinement Loop:**

For each concern identified by the simplicity review:

1. **Present the concern** using AskUserQuestion:
   ```yaml
   AskUserQuestion:
     questions:
       - question: "Simplicity Review: {brief concern title}?"
         header: "YAGNI Check"
         options:
           - label: "Apply simpler approach"
             description: "{description of the simpler alternative}"
           - label: "Keep as-is"
             description: "Record this decision and skip in future reviews"
         multiSelect: false
   ```

2. **If user selects "Apply simpler approach":**
   - Read the current epic file
   - Apply the simpler alternative by editing the relevant section
   - Confirm: "✅ Updated: {section name} with simpler approach"
   - Continue to next concern

3. **If user selects "Keep as-is":**
   - Update the `## Reviewed Decisions` table in the epic:
     ```markdown
     | {decision name} | {YAGNI concern raised} | Reviewed and accepted |
     ```
   - Confirm: "📝 Recorded: {decision} kept as-is (won't be raised again)"
   - Continue to next concern

4. **After all concerns addressed:**
   ```
   ✅ Simplicity review complete
     - Changes applied: {n}
     - Decisions kept as-is: {n}
   ```

**Skip Logic:** When running simplicity review:
- Always read the `## Reviewed Decisions` section first
- Skip any concerns that match entries in that table
- Only present genuinely new concerns

### 7. Output

```
═══════════════════════════════════════════════════════════════
  ✅ EPIC CREATED
═══════════════════════════════════════════════════════════════

  Created: workflow/epics/$ARGUMENTS/epic.md
  GitHub:  {#issue_num if promoted, otherwise "not synced"}

  Summary:
    User flows: {n} identified
    Tasks: ~{n} estimated
    Complexity: {XS/S/M/L}
    Simplicity: {n} changes applied, {n} decisions reviewed

  📍 Workflow position: Epic defined, ready to decompose into tasks

  ➡️  Next steps:
     /pm:epic-decompose $ARGUMENTS  ← Break into tasks (recommended)
     /pm:epic-edit $ARGUMENTS       ← Edit epic details
     /pm:epic-sync $ARGUMENTS       ← Sync to GitHub

  🔄 Related:
     /pm:epic-status                ← View all epics
     /pm:status                     ← Overall workflow status

═══════════════════════════════════════════════════════════════
```

## Important Notes

- **Keep epics focused** - Aim for ≤10 tasks
- **Leverage existing code** - Don't reinvent what exists
- **Identify simplifications** - Question each requirement's necessity
- **PRD promotion** - If PRD had a GitHub issue, epic inherits it (Backlog → ToDo)
- Follow `/rules/datetime.md` for timestamps
