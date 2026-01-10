---
allowed-tools: Bash, Read, Write, LS, Task, Glob, Grep
---

# Epic Decompose Detailed

Break epic into highly detailed, self-contained tasks suitable for execution by external AI agents that don't have interactive codebase access.

## Usage
```
/pm:epic-decompose-detailed <feature_name>
```

## Purpose

This variant generates task files with extensive context for external AI executors (e.g., DeepSeek via DeepInfra) that:
- Cannot explore the codebase interactively
- Don't have access to `.claude/rules/`
- Don't know existing patterns unless explicitly shown
- Will make destructive changes unless told not to
- Won't write tests unless explicitly required with examples

## Required Rules

**IMPORTANT:** Before executing this command, read and follow:
- `.claude/rules/datetime.md` - For getting real current date/time

## Phase 0: Prime Context (ALWAYS RUN FIRST)

Before any research or user interaction, load project context:

1. **Check context exists:**
   ```bash
   ls -la .claude/context/*.md 2>/dev/null | wc -l
   ```
   - If 0 files: Tell user "❌ No context found. Run /ccpmcontext:create first." and stop.

2. **Load context files in order:**
   - Read `.claude/context/project-overview.md` - Project understanding
   - Read `.claude/context/tech-context.md` - Technical stack
   - Read `.claude/context/progress.md` - Current status
   - Read `.claude/context/system-patterns.md` - Architecture patterns
   - Read `.claude/context/project-structure.md` - File organization

3. **Brief acknowledgment (don't be verbose):**
   ```
   🧠 Context loaded ({n} files). Project: {name}, Branch: {branch}
   ```

## Preflight Checklist

Before proceeding, complete these validation steps.
Do not bother the user with preflight checks progress. Just do them and move on.

1. **Verify epic exists:**
   - Check if `workflow/epics/$ARGUMENTS/epic.md` exists
   - If not found, tell user: "❌ Epic not found: $ARGUMENTS. First create it with: /pm:epic-create $ARGUMENTS"
   - Stop execution if epic doesn't exist

2. **Check for existing tasks:**
   - Check if any numbered task files (001.md, 002.md, etc.) already exist in `workflow/epics/$ARGUMENTS/`
   - If tasks exist, list them and ask: "⚠️ Found {count} existing tasks. Delete and recreate all tasks? (yes/no)"
   - Only proceed with explicit 'yes' confirmation
   - If user says no, suggest: "View existing tasks with: /pm:epic-show $ARGUMENTS"

3. **Validate epic frontmatter:**
   - Verify epic has valid frontmatter with: name, status, created, prd
   - If invalid, tell user: "❌ Invalid epic frontmatter. Please check: workflow/epics/$ARGUMENTS/epic.md"

4. **Check epic status:**
   - If epic status is already "completed", warn user: "⚠️ Epic is marked as completed. Are you sure you want to decompose it again?"

## Instructions

You are decomposing an epic into highly detailed, self-contained tasks for: **$ARGUMENTS**

### Phase 1: Codebase Pattern Analysis

**CRITICAL:** Before creating any tasks, analyze the existing codebase to understand patterns.

#### 1.1 Identify Reference Modules

Scan the codebase to find the best reference modules for this epic:

```bash
# Find existing NestJS modules
ls -la backend/src/

# Find frontend components
ls -la frontend/src/components/ frontend/src/pages/
```

For each task type (backend module, frontend component, etc.), identify:
- The closest existing module/component to use as a pattern
- Core services that must be reused (never recreated)
- Files that should never be modified or deleted

#### 1.2 Document Core Services (DO NOT RECREATE)

Identify and document services that MUST be injected, never instantiated:
- `DynamoDBService` from `@/storage/services/dynamodb.service`
- `S3Service` from `@/storage/services/s3.service`
- `LoggerService` from `@/common/logger`
- `ConfigService` from `@nestjs/config`

#### 1.3 Identify Pattern Files

For this specific epic, identify:
- Service pattern file: `backend/src/{similar-module}/services/{similar}.service.ts`
- Controller pattern file: `backend/src/{similar-module}/{similar}.controller.ts`
- DTO pattern file: `backend/src/{similar-module}/dto/{similar}.dto.ts`
- Test pattern file: `backend/src/{similar-module}/{similar}.service.spec.ts`
- Frontend component patterns if applicable

#### 1.4 Identify Protected Files

List files that tasks should NOT modify or delete:
- Core storage services: `backend/src/storage/services/*.ts`
- Auth decorators: `backend/src/auth/decorators/*.ts`
- App module (only additive changes): `backend/src/app.module.ts`
- Shared types: `packages/shared-types/src/**/*.ts`

### Phase 2: Read the Epic

- Load the epic from `workflow/epics/$ARGUMENTS/epic.md`
- Understand the technical approach and requirements
- Review the task breakdown preview
- Map each task to the patterns identified in Phase 1

### Phase 3: Create Detailed Task Files

For each task, create a file with ALL the context needed for an external agent.

#### Detailed Task File Format

```markdown
---
name: [Task Title]
status: open
created: [Current ISO date/time]
updated: [Current ISO date/time]
depends_on: []
parallel: true
conflicts_with: []
---

# Task: [Task Title]

## Description
Clear, concise description of what needs to be done

## CRITICAL CONSTRAINTS
⚠️ THESE RULES ARE MANDATORY - VIOLATION WILL CAUSE TASK FAILURE

### Files You MUST NOT Modify or Delete
- `backend/src/storage/services/dynamodb.service.ts` - DO NOT MODIFY
- `backend/src/storage/services/s3.service.ts` - DO NOT MODIFY
- `backend/src/auth/decorators/current-user.decorator.ts` - DO NOT MODIFY
- [List any other protected files specific to this task]

### Services You MUST Use (DO NOT Create New Clients)
- Use `DynamoDBService` from `@/storage` - NOT `new DynamoDB()` or `new DynamoDBClient()`
- Use `S3Service` from `@/storage` - NOT `new S3Client()`
- Use `CurrentUser` decorator from `@/auth` - NOT custom auth logic
- Use `LoggerService` from `@/common/logger` - NOT `console.log()`
- [List other required services]

### Import Patterns You MUST Follow
\`\`\`typescript
// Correct - use barrel exports
import { DynamoDBService, DBKeys } from '../../storage';
import { CurrentUser, type AuthenticatedUser } from '../../auth';
import { LoggerService } from '../../common/logger';

// WRONG - never do this
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { S3Client } from '@aws-sdk/client-s3';
\`\`\`

## REQUIRED READING
🔴 You MUST read these files BEFORE writing any code. Copy their patterns exactly.

### Pattern Files to Read First
1. `backend/src/{pattern-module}/{pattern-module}.service.ts` - Service class structure
2. `backend/src/{pattern-module}/{pattern-module}.controller.ts` - Controller decorators and routes
3. `backend/src/{pattern-module}/dto/{pattern}.dto.ts` - DTO and Zod validation patterns
4. `backend/src/{pattern-module}/{pattern-module}.service.spec.ts` - Test file structure
5. `backend/src/storage/utils/db-keys.ts` - DynamoDB key patterns

### Why This Matters
The codebase has specific patterns for:
- Dependency injection (constructor injection, not manual instantiation)
- Error handling (NestJS exceptions, not custom errors)
- Validation (Zod schemas with safeParse, not class-validator in controllers)
- Logging (LoggerService with structured logging)
Using generic patterns will break the build or create inconsistencies.

## EXACT IMPLEMENTATION TEMPLATE
Copy this structure exactly, filling in the specifics:

### File: `backend/src/{module}/{module}.module.ts`
\`\`\`typescript
import { Module } from '@nestjs/common';
import { StorageModule } from '../storage';
import { {Module}Controller } from './{module}.controller';
import { {Module}Service } from './services/{module}.service';

@Module({
  imports: [StorageModule],
  controllers: [{Module}Controller],
  providers: [{Module}Service],
  exports: [{Module}Service],
})
export class {Module}Module {}
\`\`\`

### File: `backend/src/{module}/{module}.controller.ts`
\`\`\`typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  HttpCode,
  HttpStatus,
  BadRequestException,
} from '@nestjs/common';
import { CurrentUser, type AuthenticatedUser } from '../auth';
import { {Module}Service } from './services/{module}.service';
import { Create{Module}Schema, type Create{Module}Response } from './dto/{module}.dto';

@Controller('api/v1')
export class {Module}Controller {
  constructor(private readonly {module}Service: {Module}Service) {}

  @Post('{module}s')
  @HttpCode(HttpStatus.CREATED)
  async create(
    @CurrentUser() user: AuthenticatedUser,
    @Body() body: unknown,
  ): Promise<Create{Module}Response> {
    const parsed = Create{Module}Schema.safeParse(body);
    if (!parsed.success) {
      throw new BadRequestException(parsed.error.flatten().fieldErrors);
    }
    return this.{module}Service.create(user.userId, parsed.data);
  }
}
\`\`\`

### File: `backend/src/{module}/services/{module}.service.ts`
\`\`\`typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { DynamoDBService, DBKeys } from '../../storage';
import { LoggerService } from '../../common/logger';
import type { Create{Module}Dto, {Module}Response } from '../dto/{module}.dto';
import { randomUUID } from 'crypto';

@Injectable()
export class {Module}Service {
  private readonly logger = new LoggerService('{Module}Service');

  constructor(
    private readonly dynamodb: DynamoDBService,  // INJECT, don't instantiate
  ) {}

  async create(userId: string, dto: Create{Module}Dto): Promise<{Module}Response> {
    const id = randomUUID();
    const now = new Date().toISOString();

    const item = {
      pk: DBKeys.{module}(id),  // Add this key pattern to db-keys.ts
      sk: 'META',
      id,
      userId,
      ...dto,
      createdAt: now,
    };

    await this.dynamodb.putItem(item);
    this.logger.log(\`{Module} \${id} created by user \${userId}\`);

    return { id, ...dto, createdAt: now };
  }
}
\`\`\`

### File: `backend/src/{module}/dto/{module}.dto.ts`
\`\`\`typescript
import { z } from 'zod';

export const Create{Module}Schema = z.object({
  name: z.string().min(1).max(255),
  // Add other fields following this pattern
});

export type Create{Module}Dto = z.infer<typeof Create{Module}Schema>;

export interface {Module}Response {
  id: string;
  name: string;
  createdAt: string;
}
\`\`\`

## REQUIRED TESTS
🔴 Tests are MANDATORY. Task is NOT complete without passing tests.

### Test File: `backend/src/{module}/services/{module}.service.spec.ts`
\`\`\`typescript
import { Test, TestingModule } from '@nestjs/testing';
import { {Module}Service } from './{module}.service';
import { DynamoDBService, DBKeys } from '../../storage';

describe('{Module}Service', () => {
  let service: {Module}Service;
  let dynamoDBService: jest.Mocked<DynamoDBService>;

  const mockUserId = 'user_123';

  beforeEach(async () => {
    const mockDynamoDBService = {
      putItem: jest.fn(),
      getItem: jest.fn(),
      queryByPK: jest.fn(),
      updateItem: jest.fn(),
      deleteItem: jest.fn(),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        {Module}Service,
        { provide: DynamoDBService, useValue: mockDynamoDBService },
      ],
    }).compile();

    service = module.get<{Module}Service>({Module}Service);
    dynamoDBService = module.get(DynamoDBService);
  });

  describe('create', () => {
    it('should create a {module} and return it', async () => {
      dynamoDBService.putItem.mockResolvedValue(undefined);

      const dto = { name: 'Test {Module}' };
      const result = await service.create(mockUserId, dto);

      expect(result.name).toBe('Test {Module}');
      expect(result.id).toBeDefined();
      expect(result.createdAt).toBeDefined();
      expect(dynamoDBService.putItem).toHaveBeenCalledWith(
        expect.objectContaining({
          pk: expect.stringContaining('{MODULE}#'),
          sk: 'META',
          name: 'Test {Module}',
          userId: mockUserId,
        }),
      );
    });
  });

  // Add more tests for each method
});
\`\`\`

### Minimum Test Coverage Required
- [ ] Service: create, getById, list methods tested
- [ ] Each method: success case and error case
- [ ] Mocks verify correct DBKeys usage

## FILE CHECKLIST
All files below MUST be created. Check off as you complete:

### Files to Create
- [ ] `backend/src/{module}/{module}.module.ts`
- [ ] `backend/src/{module}/{module}.controller.ts`
- [ ] `backend/src/{module}/services/{module}.service.ts`
- [ ] `backend/src/{module}/services/{module}.service.spec.ts` ← TEST FILE REQUIRED
- [ ] `backend/src/{module}/dto/{module}.dto.ts`
- [ ] `backend/src/{module}/index.ts`

### Files to Modify (ADDITIVE ONLY - DO NOT DELETE EXISTING CODE)
- [ ] `backend/src/app.module.ts` - Add import and module to imports array
- [ ] `backend/src/storage/utils/db-keys.ts` - Add key pattern (if new entity type)

### Verification Commands
After implementation, run these commands to verify:
\`\`\`bash
# Build check
cd backend && pnpm build

# Run tests for this module
cd backend && pnpm test -- --testPathPattern="{module}"

# Lint check
cd backend && pnpm lint
\`\`\`

## Acceptance Criteria
- [ ] Specific criterion 1
- [ ] Specific criterion 2
- [ ] Specific criterion 3

## Dependencies
- [ ] Task/Issue dependencies
- [ ] External dependencies

## Effort Estimate
- Size: XS/S/M/L/XL
- Hours: estimated hours
- Parallel: true/false

## Definition of Done
- [ ] All files in FILE CHECKLIST created
- [ ] All tests written and passing
- [ ] Build passes without errors
- [ ] Lint passes without errors
- [ ] All CRITICAL CONSTRAINTS followed
```

### Phase 4: Task Analysis for Pattern Selection

When creating each task, perform this analysis:

#### For Backend Tasks:
1. Identify the most similar existing module:
   ```bash
   ls backend/src/
   # Look for modules with similar functionality
   ```

2. Read that module's structure:
   ```bash
   ls -la backend/src/{similar-module}/
   cat backend/src/{similar-module}/{similar}.controller.ts
   cat backend/src/{similar-module}/services/{similar}.service.ts
   ```

3. Use those exact patterns in the task template

#### For Frontend Tasks:
1. Identify similar components:
   ```bash
   ls frontend/src/components/
   ls frontend/src/pages/
   ```

2. Read the component patterns:
   ```bash
   cat frontend/src/components/{Similar}/{Similar}.tsx
   cat frontend/src/components/{Similar}/{Similar}.test.tsx
   ```

3. Document the patterns in REQUIRED READING section

### Phase 5: Parallel Task Creation

If tasks can be created in parallel, spawn sub-agents:

```yaml
Task:
  description: "Create detailed task files batch {X}"
  subagent_type: "general-purpose"
  prompt: |
    Create detailed task files for epic: $ARGUMENTS

    Pattern Analysis Results:
    - Reference module: {identified module}
    - Protected files: {list}
    - Required services: {list}

    Tasks to create:
    - {list of 2-3 tasks for this batch}

    Use the detailed format with all sections:
    - CRITICAL CONSTRAINTS
    - REQUIRED READING
    - EXACT IMPLEMENTATION TEMPLATE
    - REQUIRED TESTS
    - FILE CHECKLIST

    Return: List of files created
```

### Phase 6: Task Naming and Structure

Save tasks as: `workflow/epics/$ARGUMENTS/{task_number}.md`
- Use sequential numbering: 001.md, 002.md, etc.
- Keep task titles short but descriptive

### Phase 7: Update Epic with Task Summary

After creating all tasks, update the epic file by adding:
```markdown
## Tasks Created (Detailed Format)
- [ ] 001.md - {Task Title} (parallel: true/false)
- [ ] 002.md - {Task Title} (parallel: true/false)
- etc.

Total tasks: {count}
Format: Detailed (external-agent compatible)
Pattern Reference: {identified reference module}
Protected Files: {count}
```

### Phase 8: Quality Validation

Before finalizing tasks, verify each task has:
- [ ] CRITICAL CONSTRAINTS section with specific files
- [ ] REQUIRED READING section with actual file paths (verified to exist)
- [ ] EXACT IMPLEMENTATION TEMPLATE with code from reference module
- [ ] REQUIRED TESTS section with test template
- [ ] FILE CHECKLIST with all files to create/modify
- [ ] Acceptance criteria are specific and testable
- [ ] Task sizes are reasonable (1-3 days each)

### Phase 9: Setup Worktree for External Agent

Create the worktree so external agents can work in isolation:

```bash
# Ensure staging is current
git checkout staging
git pull origin staging

# Create worktree directory and branch
mkdir -p .worktrees/epic
git worktree add .worktrees/epic/$ARGUMENTS -b epic/$ARGUMENTS

# Copy dependencies (faster than fresh install)
WORKTREE_PATH=.worktrees/epic/$ARGUMENTS
cp -r backend/node_modules $WORKTREE_PATH/backend/
cp -r frontend/node_modules $WORKTREE_PATH/frontend/
cp -r packages/shared-types/node_modules $WORKTREE_PATH/packages/shared-types/
cp -r packages/shared-types/dist $WORKTREE_PATH/packages/shared-types/

# Copy .env files
cp backend/.env $WORKTREE_PATH/backend/.env
cp frontend/.env $WORKTREE_PATH/frontend/.env

echo "✅ Worktree ready: $WORKTREE_PATH"
```

### Phase 10: Output External Agent Prompt

After worktree setup, output this prompt for the external AI agent:

```markdown
═══════════════════════════════════════════════════════════════════════════════
  EXTERNAL AGENT EXECUTION PROMPT
═══════════════════════════════════════════════════════════════════════════════

## Worktree Path
.worktrees/epic/$ARGUMENTS

## Epic Overview
Read `workflow/epics/$ARGUMENTS/epic.md` first. This file contains:
- **Dependency graph**: Tasks are numbered (001.md, 002.md...) with `depends_on` fields
- **Implementation order**: Follow the dependency chain - don't start a task until its dependencies are complete
- **Parallel flags**: Tasks with `parallel: true` and no dependencies can run simultaneously

## How to Work with Task Files

1. **Find your task**: `workflow/epics/$ARGUMENTS/{task_num}.md`
2. **Check dependencies**: Look at `depends_on:` in frontmatter - those must be `status: closed` first
3. **Read CRITICAL CONSTRAINTS**: These are non-negotiable rules
4. **Read REQUIRED READING files**: Understand patterns before coding
5. **Follow the template**: Use EXACT IMPLEMENTATION TEMPLATE as your starting point
6. **Write tests**: Tests are MANDATORY - use REQUIRED TESTS section
7. **Run verification**: Execute commands in Verification Commands section
8. **Update status**: Change frontmatter `status: open` → `status: closed` when done

## Agent Coordination Rules (Essential)

**Commit Often**
- Small, focused commits: `Task 001: Add user service`
- Commit after each logical unit of work

**Stay in Your Lane**
- Only modify files listed in your task's FILE CHECKLIST
- Never touch files in another task's scope without coordination

**Protected Files**
- NEVER modify files listed in CRITICAL CONSTRAINTS
- Inject services, don't instantiate them

**On Completion**
1. All tests pass: `cd backend && pnpm test`
2. Build succeeds: `cd backend && pnpm build`
3. Lint passes: `cd backend && pnpm lint`
4. Update task frontmatter: `status: closed`
5. Commit: `git add . && git commit -m "Task {num}: Complete {name}"`

**If Blocked**
- Don't proceed with incomplete dependencies
- Document the blocker in task file
- Move to another independent task

## Quick Start
```bash
cd .worktrees/epic/$ARGUMENTS
cat workflow/epics/$ARGUMENTS/epic.md           # Understand the epic
cat workflow/epics/$ARGUMENTS/001.md            # Read first task
# Follow task instructions...
```

═══════════════════════════════════════════════════════════════════════════════
```

### Phase 11: Summary Output

After all phases complete:

```
✅ Created {count} detailed tasks for epic: $ARGUMENTS

Summary:
  Reference module: {identified module}
  Protected files: {count}
  Test files required: {count}
  Worktree: .worktrees/epic/$ARGUMENTS

Task List:
  001.md - {name} (parallel: {true/false})
  002.md - {name} (parallel: {true/false})
  ...

External agent prompt output above ↑

Next steps:
  1. Copy the external agent prompt to your AI agent
  2. Or sync to GitHub: /pm:epic-sync $ARGUMENTS
  3. Or start orchestration: /pm:epic-start $ARGUMENTS
```

## Error Recovery

If any step fails:
- If task creation partially completes, list which tasks were created
- Provide option to clean up partial tasks
- Never leave the epic in an inconsistent state

## Comparison with Standard Decompose

| Feature | epic-decompose | epic-decompose-detailed |
|---------|---------------|------------------------|
| Pattern analysis | No | Yes - scans codebase |
| Protected files list | No | Yes - per task |
| Implementation templates | Basic | Full code templates |
| Test requirements | Mentioned | Complete test file template |
| File checklist | No | Yes - with modify vs create |
| External agent compatible | No | Yes - fully self-contained |
| Creation time | Fast | Slower (thorough analysis) |

Use this command when tasks will be executed by external AI agents that lack interactive codebase access.
