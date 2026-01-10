# Simplify After Plan Rule

After planning or prototyping an implementation, step back and look for simpler approaches before committing to the full implementation.

## The Pattern

```
Plan/Prototype → Step Back → Simplify → Implement
```

This mirrors how experienced developers work:
1. Sketch out a solution (plan or prototype)
2. See the whole picture
3. Ask: "Can I achieve the same outcome more simply?"
4. Implement the simpler version

## When to Apply

Apply this rule after:
- Creating an epic with technical approach
- Writing a prototype or spike
- Planning a multi-file change
- Designing a new component/service

## Questions to Ask

Before finalizing a plan, ask:

1. **Can I remove something?**
   - Is every file/component necessary?
   - Can two things become one?
   - Is there unused flexibility?

2. **Can I use what already exists?**
   - Does a pattern already exist in the codebase?
   - Can I extend rather than create?
   - Is there a simpler library function?

3. **Am I solving future problems?**
   - Is this solving today's requirement or a hypothetical one?
   - Can I defer complexity until it's actually needed?
   - Am I adding "just in case" code?

4. **What's the smallest change?**
   - What's the minimum code to achieve the goal?
   - Can I avoid touching certain files entirely?
   - Is there a one-liner that replaces my multi-file approach?

## Integration with PM Workflow

### In `/pm:epic-create`
After drafting the technical approach, run simplification review:
- Check for over-engineering
- Look for existing patterns to reuse
- Ask if scope can be reduced

### In `/pm:epic-decompose`
Before finalizing tasks:
- Can tasks be combined?
- Are there tasks that could be eliminated?
- Is the architecture simpler than proposed?

### After Prototyping
When you've written exploratory code:
- What did you learn?
- What can be removed?
- What's the minimal version that works?

## Examples

### Before Simplification
```
Plan: Create new AuthService, AuthGuard, AuthMiddleware,
      AuthContext, useAuth hook, auth utilities...
```

### After Stepping Back
```
Simplified: Extend existing FirebaseAuthGuard with one new method,
           add single utility function to existing auth utils.
```

### The Question That Helps
> "If I had to implement this in half the files, what would I cut?"

## Boy Scout Rule Integration

When working on a file for your main task:
- Notice adjacent code that could be cleaner
- Make small improvements if they're safe
- Keep cleanup commits separate from feature commits
- Don't refactor unrelated code

## Anti-Patterns to Avoid

❌ Implementing the first solution that comes to mind
❌ Adding abstractions "for future flexibility"
❌ Creating new patterns when existing ones work
❌ Touching files you don't need to touch
❌ Adding configuration for things that won't change

## Remember

> "The best code is no code at all. The second best is simple code."

Every line you don't write is a line you don't have to:
- Test
- Debug
- Maintain
- Explain
- Update when dependencies change
