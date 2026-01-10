# Playwright Testing Modes

This project uses Playwright for E2E testing with two distinct modes. Choose the right mode for your testing scenario.

## Mode Selection Guide

| Testing Scenario | Mode | Why |
|------------------|------|-----|
| Public pages, UI flows | Chromium | Fast, isolated, no setup |
| Chrome extension features | Google Profile | Extension must be installed |
| Firebase/Google auth | Google Profile | Real auth cookies needed |
| CI/CD pipelines | Chromium (headless) | No GUI available |
| Debugging locally | Either + `--debug` | Interactive debugging |

## Mode 1: Attached Chromium (Default)

Standard Playwright mode. Launches isolated Chromium instance.

```bash
npx playwright test
npx playwright test --ui  # With inspector
```

**Characteristics:**
- Clean browser state each run
- Fast startup
- Works in CI/CD
- No real authentication
- No extensions

## Mode 2: Google Profile

Connects to existing Chrome with your Google profile via CDP (Chrome DevTools Protocol).

**Setup required:**

```bash
# 1. Close all Chrome windows first!

# 2. Start Chrome with remote debugging (Linux)
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.config/google-chrome"

# 3. Verify connection
curl http://localhost:9222/json/version
```

**Characteristics:**
- Uses your real Chrome profile
- Has your extensions installed
- Has your auth cookies (Firebase, Google)
- Slower startup
- Cannot run in headless CI

## When to Ask User About Mode

Ask about mode when:
1. Test file mentions `extension`, `chrome.runtime`, or `chrome.tabs`
2. Test requires authenticated state and no mock is available
3. User mentions "test the extension"
4. Test is failing in Chromium mode with auth-related errors

## Code Patterns

### Chromium Mode (playwright.config.ts)
```typescript
export default defineConfig({
  use: {
    browserName: 'chromium',
    headless: false,
  },
});
```

### Google Profile Mode (test file)
```typescript
import { chromium, Browser, Page } from 'playwright';

let browser: Browser;
let page: Page;

test.beforeAll(async () => {
  browser = await chromium.connectOverCDP('http://localhost:9222');
  const context = browser.contexts()[0];
  page = context.pages()[0] || await context.newPage();
});
```

## Common Issues

### "Extension not found" in tests
→ Using Chromium mode instead of Google Profile mode

### "Not authenticated" errors
→ Need Google Profile mode with logged-in Chrome

### "Cannot connect to port 9222"
→ Chrome not started with `--remote-debugging-port=9222`

### Tests pass locally, fail in CI
→ Using Google Profile mode locally but CI uses Chromium
→ Add mock auth for CI or skip extension tests

## Quick Commands

```bash
# Standard E2E
cd frontend && npx playwright test

# With UI mode (debugging)
cd frontend && npx playwright test --ui

# Specific test file
cd frontend && npx playwright test auth.spec.ts

# Debug mode (step through)
cd frontend && npx playwright test --debug

# Show last report
cd frontend && npx playwright show-report
```

## Integration with PM Workflow

The `/pm:epic-verify` command runs E2E tests. It should:
1. Check if any tests require extension/auth
2. Prompt for mode if needed
3. Guide user through Chrome setup for Profile mode
4. Run tests and report results

See also: `/testing:e2e` command for interactive mode selection.
