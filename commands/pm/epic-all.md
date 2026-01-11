---
allowed-tools: Bash, Read, Write, LS, Skill
---

# Epic All

Run the complete epic workflow from creation to close autonomously. No user interaction required unless a critical failure occurs.

## Usage
```
/pm:epic-all <feature_name>
```

## What This Command Does

Orchestrates the full epic lifecycle **autonomously**:

```
epic-create → epic-decompose → epic-sync → epic-start → epic-verify → epic-close
```

- Runs without user interaction
- Handles minor issues autonomously
- Only stops on critical failures (verification fails, merge conflicts)
- Resumes from current position if interrupted

## Phase Detection

Before running any step, detect current epic state to resume from the right point:

```bash
EPIC_NAME="$ARGUMENTS"
EPIC_DIR="workflow/epics/$EPIC_NAME"

# Determine current state
if [ ! -d "$EPIC_DIR" ]; then
  CURRENT_PHASE="create"
elif [ ! -f "$EPIC_DIR/epic.md" ]; then
  CURRENT_PHASE="create"
else
  # Epic exists - check for tasks
  task_count=$(ls "$EPIC_DIR"/[0-9][0-9][0-9].md 2>/dev/null | wc -l)

  if [ "$task_count" -eq 0 ]; then
    CURRENT_PHASE="decompose"
  else
    # Tasks exist - check status
    epic_status=$(grep '^status:' "$EPIC_DIR/epic.md" | sed 's/status: *//')
    verified=$(grep '^verified:' "$EPIC_DIR/epic.md" 2>/dev/null | sed 's/verified: *//')

    if [ "$epic_status" = "completed" ]; then
      CURRENT_PHASE="done"
    elif [ "$verified" = "true" ]; then
      CURRENT_PHASE="close"
    elif [ "$epic_status" = "in-progress" ]; then
      # Check if all tasks closed
      open_tasks=$(grep -l '^status: open' "$EPIC_DIR"/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
      in_progress=$(grep -l '^status: in-progress' "$EPIC_DIR"/[0-9][0-9][0-9].md 2>/dev/null | wc -l)

      if [ "$open_tasks" -eq 0 ] && [ "$in_progress" -eq 0 ]; then
        CURRENT_PHASE="verify"
      else
        CURRENT_PHASE="start"
      fi
    else
      # Backlog with tasks = needs sync then start
      github=$(grep '^github:' "$EPIC_DIR/epic.md" 2>/dev/null | sed 's/github: *//')
      if [ -z "$github" ] || [ "$github" = "" ]; then
        CURRENT_PHASE="sync"
      else
        CURRENT_PHASE="start"
      fi
    fi
  fi
fi

echo "Current phase: $CURRENT_PHASE"
```

## Workflow Phases

### Phase 0: Already Complete

```bash
if [ "$CURRENT_PHASE" = "done" ]; then
  echo "✅ Epic already completed: $EPIC_NAME"
  echo ""
  echo "Epic file: $EPIC_DIR/epic.md"
  exit 0
fi
```

### Phase 1: Create Epic

```bash
if [ "$CURRENT_PHASE" = "create" ]; then
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  PHASE 1: CREATE EPIC"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
fi
```

```yaml
# Only run if in create phase
Skill:
  skill: "pm:epic-create"
  args: "$ARGUMENTS"
```

Wait for completion, then continue to next phase.

### Phase 2: Decompose Epic

```bash
if [ "$CURRENT_PHASE" = "create" ] || [ "$CURRENT_PHASE" = "decompose" ]; then
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  PHASE 2: DECOMPOSE INTO TASKS"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
fi
```

```yaml
Skill:
  skill: "pm:epic-decompose"
  args: "$ARGUMENTS"
```

Note: For external agent execution (Llama, DeepSeek, etc.), use `/pm:epic-decompose-detailed` separately.

### Phase 3: Sync to GitHub

```bash
if [ "$CURRENT_PHASE" = "sync" ]; then
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  PHASE 3: SYNC TO GITHUB"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
fi
```

```yaml
Skill:
  skill: "pm:epic-sync"
  args: "$ARGUMENTS"
```

### Phase 4: Start Epic Execution

```bash
if [ "$CURRENT_PHASE" = "create" ] || [ "$CURRENT_PHASE" = "decompose" ] ||
   [ "$CURRENT_PHASE" = "sync" ] || [ "$CURRENT_PHASE" = "start" ]; then
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  PHASE 4: START EPIC EXECUTION"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
fi
```

```yaml
Skill:
  skill: "pm:epic-start"
  args: "$ARGUMENTS"
```

**Note:** epic-start orchestrates task execution. This is where the bulk of the work happens.
When all tasks are closed, continue to verification.

### Phase 5: Verify Epic

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  PHASE 5: VERIFY (RUN TESTS)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

```yaml
Skill:
  skill: "pm:epic-verify"
  args: "$ARGUMENTS"
```

If verification fails, stop and report:

```bash
verified=$(grep '^verified:' workflow/epics/$ARGUMENTS/epic.md 2>/dev/null | sed 's/verified: *//')
if [ "$verified" != "true" ]; then
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  ⚠️  VERIFICATION FAILED"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""
  echo "  Tests failed. Fix issues and re-run:"
  echo "    /pm:epic-all $ARGUMENTS"
  echo ""
  echo "  Or run verification alone:"
  echo "    /pm:epic-verify $ARGUMENTS"
  echo ""
  exit 1
fi
```

### Phase 6: Close Epic

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  PHASE 6: CLOSE EPIC"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

```yaml
Skill:
  skill: "pm:epic-close"
  args: "$ARGUMENTS"
```

## Final Output

```
═══════════════════════════════════════════════════════════════════════════════
  ✅ EPIC WORKFLOW COMPLETE: $ARGUMENTS
═══════════════════════════════════════════════════════════════════════════════

  Phases completed:
    1. ✅ Created epic
    2. ✅ Decomposed into {n} tasks
    3. ✅ Synced to GitHub
    4. ✅ Executed all tasks
    5. ✅ Verified (tests passed)
    6. ✅ Closed (merged to staging)

  Summary:
    • Tasks completed: {n}
    • Duration: {time}
    • Branch: epic/$ARGUMENTS → staging

  Next steps:
    • Deploy staging to test environment
    • When ready: merge staging → main for production
    • Start new work: /pm:epic-all <new-feature>

═══════════════════════════════════════════════════════════════════════════════
```

## Error Handling

### Phase Errors

Each phase skill handles its own errors. If a phase fails:

1. **epic-create fails:** Report error, suggest `/pm:epic-create` to retry
2. **epic-decompose fails:** Report error, epic exists but no tasks
3. **epic-sync fails:** Continue (warning only - GitHub is optional)
4. **epic-start fails:** Tasks may be partially complete, can resume
5. **epic-verify fails:** Stop, report which tests failed, suggest fixes
6. **epic-close fails:** Usually merge conflict - provide resolution steps

### Resuming After Failure

The phase detection logic automatically resumes from the current state:

```bash
# Just re-run the same command
/pm:epic-all $ARGUMENTS
```

The command will:
- Skip already-completed phases
- Resume from the failed/incomplete phase
- Continue to the end

## Important Notes

- **PRD First?** If you have a PRD, run `/pm:prd-new` before this command.
  epic-create will pick up the PRD automatically.
- **Fully Autonomous:** Runs without user interaction. Only stops on critical failures.
- **Long Running:** epic-start orchestrates task execution which can take significant time.
- **GitHub Optional:** If `.claude/project.yaml` isn't configured, GitHub sync is skipped.
- **Merge to Staging:** epic-close merges to staging, not main. Deploy to production separately.
- **External Agents:** For execution via external models (Llama, DeepSeek), use
  `/pm:epic-decompose-detailed` instead of this command.
