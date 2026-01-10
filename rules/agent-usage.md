# Agent Usage Rule

When to use specialized agents from compound-engineering for enhanced quality.

## Available Agents

### Research Agents

**framework-docs-researcher** (Context7 MCP)
- **Use when:** Need official framework/library documentation
- **Triggered in:** `/pm:prd-new`, `/pm:epic-create`, `/pm:bugfix`
- **Purpose:** Fetch latest docs from React, NestJS, and other dependencies

**best-practices-researcher** (Context7 MCP)
- **Use when:** Need industry best practices for a pattern
- **Purpose:** Research common solutions, anti-patterns to avoid

### Review Agents

**code-simplicity-reviewer** (YAGNI)
- **Use when:** After creating epics or completing bug fixes
- **Triggered in:** `/pm:epic-create` (Step 6), `/pm:bugfix` (Phase 7.5)
- **Purpose:** Identify over-engineering, premature abstractions
- **Action:** Present concerns to user for "apply simpler" or "keep as-is" decision

**architecture-strategist**
- **Use when:** Before decomposing epics into tasks
- **Triggered in:** `/pm:epic-decompose` (Step 1.5)
- **Purpose:** Validate component boundaries, identify parallel opportunities

## Integration Points

| Command | Agents Used | Purpose |
|---------|-------------|---------|
| `/pm:prd-new` | framework-docs-researcher | Research patterns |
| `/pm:epic-create` | framework-docs-researcher, code-simplicity-reviewer | Research + YAGNI review |
| `/pm:epic-decompose` | architecture-strategist | Validate boundaries |
| `/pm:bugfix` | framework-docs-researcher, code-simplicity-reviewer | Research + fix review |

## YAGNI Review Process

When code-simplicity-reviewer identifies concerns:

1. **For each concern:**
   - Present via `AskUserQuestion` with two options:
     - "Apply simpler approach" - implement the suggestion
     - "Keep as-is" - record decision in Reviewed Decisions table

2. **Skip logic:**
   - Check `## Reviewed Decisions` section first
   - Don't re-raise concerns already reviewed

3. **Documentation:**
   ```markdown
   ## Reviewed Decisions

   | Decision | Concern Raised | Rationale for Keeping |
   |----------|----------------|----------------------|
   | Retry logic | Adds complexity | Needed for reliability |
   ```

## When NOT to Use Agents

- **Simple file edits**: Direct editing is faster
- **Clear requirements**: No need to research what's already known
- **Time-sensitive fixes**: Skip review for hotfixes (document why)

## Manual Invocation

Agents can be invoked manually via Task tool:

```yaml
Task:
  subagent_type: "general-purpose"
  prompt: |
    As a code-simplicity-reviewer, analyze: {file or changes}
    Check for YAGNI violations and over-engineering.
    Return: Concerns with simpler alternatives.
```

## Important Notes

- Agents use Context7 MCP when available for latest docs
- Review concerns are suggestions, not mandates
- User decision is final - document and move on
- Don't block workflow on agent failures
