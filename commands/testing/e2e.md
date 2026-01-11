---
allowed-tools: Bash, Read, Write, Task, mcp__playwright-cdp__browser_navigate, mcp__playwright-cdp__browser_snapshot, mcp__playwright-cdp__browser_click, mcp__playwright-cdp__browser_take_screenshot, mcp__playwright-cdp__browser_type, mcp__playwright-cdp__browser_console_messages, mcp__playwright-cdp__browser_network_requests
---

# E2E Testing with Playwright

Run end-to-end tests with Playwright. Supports two modes:
- **Attached Chromium**: Standard mode, isolated browser instance
- **Google Profile (CDP)**: Connects to existing Chrome with auth and extensions

## IMPORTANT: MCP Tool Selection

**For Google Profile mode, use `mcp__playwright-cdp__*` tools** (CDP = Chrome DevTools Protocol).

Do NOT use `mcp__playwright__*` tools for profile mode - those launch an isolated browser that won't have your auth or extensions.

| Mode | MCP Tools | Why |
|------|-----------|-----|
| Chromium (isolated) | `mcp__playwright__*` | Launches fresh browser |
| Google Profile | `mcp__playwright-cdp__*` | Connects to existing Chrome |

## Prefer Real Calls Over Mocks

**E2E tests should use real API calls, not mocks.** See `/rules/testing-philosophy.md`.

**Exception:** Mock expensive external calls (>$1/run) like paid LLM APIs. Document why when you do.

## Usage
```
/testing:e2e [test-pattern]
```

**Examples:**
```bash
/testing:e2e                    # Run all E2E tests
/testing:e2e auth               # Run auth-related tests
/testing:e2e --extension        # Test Chrome extension (needs Google Profile mode)
```

## Mode Selection

Use `AskUserQuestion` for interactive mode selection.

```bash
# Or specify explicitly
/testing:e2e --mode=chromium
/testing:e2e --mode=profile
```

## Mode 1: Attached Chromium (Default)

Standard Playwright mode. Opens isolated Chromium instance.

**Use when:**
- Testing public pages
- Testing with mock authentication
- Running in CI/CD
- Speed is priority

**Run:**
```bash
cd frontend
npx playwright test
# or with UI
npx playwright test --ui
```

## Mode 2: Google Profile (Auth + Extensions)

Connects to existing Chrome instance via CDP (Chrome DevTools Protocol).

**Use when:**
- Testing Chrome extension functionality
- Testing with real Google/Firebase authentication
- Testing features requiring logged-in state
- Manual E2E verification of deployed apps

### The Profile Problem

Chrome cannot share its default profile directory while also running with remote debugging. You have two options:

**Option A: Copy profile to temp location (RECOMMENDED)**
```bash
# 1. Close Chrome completely
pkill -f chrome

# 2. Copy your profile (adjust profile name - check chrome://version for "Profile Path")
mkdir -p /tmp/chrome-debug
cp -r "$HOME/.config/google-chrome/Default" /tmp/chrome-debug/

# 3. Start Chrome with the copied profile
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="/tmp/chrome-debug"
```

**Option B: Close Chrome entirely and use original**
```bash
# 1. Close ALL Chrome windows and processes
pkill -f chrome

# 2. Start Chrome with debugging (will be only Chrome instance)
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.config/google-chrome"
```

### Verify Connection

```bash
curl -s http://localhost:9222/json/version | jq .
# Should show Browser, webSocketDebuggerUrl, etc.
```

### Using CDP MCP Tools

Once Chrome is running with debugging, use the CDP tools:

```bash
# Take a snapshot of current page
mcp__playwright-cdp__browser_snapshot

# Navigate
mcp__playwright-cdp__browser_navigate url="https://example.com"

# Click (use ref from snapshot)
mcp__playwright-cdp__browser_click element="Submit button" ref="submit-btn-ref"
```

### Programmatic Test Configuration

```typescript
// playwright.config.ts or test file
import { chromium } from 'playwright';

const browser = await chromium.connectOverCDP('http://localhost:9222');
const context = browser.contexts()[0];  // Use existing context
const page = context.pages()[0] || await context.newPage();
```

## Instructions

### 1. Determine Mode

Check if test requires authentication or extension:

```bash
grep -l "extension\|chrome\|auth\|login\|firebase" frontend/e2e/*.spec.ts
```

If matches found or user specified `--extension`, use Google Profile mode.

### 2. Interactive Mode Selection

Use `AskUserQuestion` tool:

```yaml
AskUserQuestion:
  questions:
    - question: "Which Playwright mode should we use?"
      header: "Test mode"
      options:
        - label: "Chromium (isolated)"
          description: "Standard mode, fast, works in CI"
        - label: "Google Profile (CDP)"
          description: "Uses existing Chrome with auth and extensions"
```

### 3. Setup for Google Profile Mode

If using Google Profile mode, guide user through setup:

```
📍 Google Profile Mode Setup

1. Close all Chrome windows: pkill -f chrome

2. Copy your profile (keeps original Chrome usable later):
   mkdir -p /tmp/chrome-debug
   cp -r "$HOME/.config/google-chrome/Default" /tmp/chrome-debug/

3. Start Chrome with debugging:
   google-chrome --remote-debugging-port=9222 --user-data-dir="/tmp/chrome-debug"

4. Log in to required services if needed (Firebase, Google, etc.)
```

Confirm Chrome is ready:

```yaml
AskUserQuestion:
  questions:
    - question: "Is Chrome running with debugging port 9222?"
      header: "Chrome ready"
      options:
        - label: "Yes, Chrome is ready"
          description: "Continue with tests"
        - label: "No, need help"
          description: "Show troubleshooting"
```

Verify:
```bash
curl -s http://localhost:9222/json/version > /dev/null && echo "✅ Connected to Chrome" || echo "❌ Cannot connect"
```

### 4. Run Tests

```bash
cd frontend

if [ "$USE_PROFILE" = true ]; then
  echo "🎭 Running with Google Profile (CDP)..."
  PLAYWRIGHT_CHROMIUM_USE_CDP=true \
  CDP_ENDPOINT=http://localhost:9222 \
  npx playwright test $TEST_PATTERN --reporter=list
else
  echo "🎭 Running with attached Chromium..."
  npx playwright test $TEST_PATTERN --reporter=list
fi
```

### 5. Handle Failures

If tests fail, offer options via `AskUserQuestion`:
- View report: `npx playwright show-report`
- Debug: `npx playwright test --debug $FAILED_TEST`
- Screenshot: Use `mcp__playwright-cdp__browser_take_screenshot`

## Quick Reference

| Scenario | Mode | MCP Tools |
|----------|------|-----------|
| Basic UI tests | Chromium | `mcp__playwright__*` |
| Auth/extension tests | Profile (CDP) | `mcp__playwright-cdp__*` |
| CI/CD | Chromium (headless) | N/A (use npx playwright) |
| Manual verification | Profile (CDP) | `mcp__playwright-cdp__*` |

## Troubleshooting

### "Cannot connect to Chrome on port 9222"
1. Close ALL Chrome: `pkill -f chrome`
2. Copy profile: `cp -r ~/.config/google-chrome/Default /tmp/chrome-debug/`
3. Start with debugging: `google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug`
4. Verify: `curl http://localhost:9222/json/version`

### "about:blank" or no auth in browser
- You're using `mcp__playwright__*` tools (wrong!)
- Switch to `mcp__playwright-cdp__*` tools for profile mode

### Extension not visible
- Ensure extension is installed in the copied profile
- Check `chrome://extensions` - must be enabled
- May need to copy specific profile (e.g., "Profile 1" not "Default")

### "No contexts available"
- Open at least one tab in Chrome before connecting
- CDP needs an existing context to attach to
