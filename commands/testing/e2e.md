---
allowed-tools: Bash, Read, Write, Task, mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_click, mcp__playwright__browser_take_screenshot
---

# E2E Testing with Playwright

Run end-to-end tests with Playwright. Supports two modes:
- **Attached Chromium**: Standard mode, isolated browser instance
- **Google Profile**: Uses existing Chrome profile with auth and extensions

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

If gum is installed, you'll get an interactive mode selector. Otherwise, specify mode via argument.

```bash
# Interactive (with gum)
MODE=$(gum choose --header "Select Playwright mode:" \
  "Attached Chromium (isolated, fast)" \
  "Google Profile (auth + extensions)")

# Or specify explicitly
/testing:e2e --mode=chromium
/testing:e2e --mode=profile
```

## Mode Details

### Mode 1: Attached Chromium (Default)

Standard Playwright mode. Opens isolated Chromium instance.

**Use when:**
- Testing public pages
- Testing with mock authentication
- Running in CI/CD
- Speed is priority

**Configuration:**
```typescript
// playwright.config.ts (default)
use: {
  browserName: 'chromium',
  headless: false,  // Set true for CI
}
```

**Run:**
```bash
npx playwright test
# or with UI
npx playwright test --ui
```

### Mode 2: Google Profile (Auth + Extensions)

Connects to existing Chrome instance with your Google profile.

**Use when:**
- Testing Chrome extension functionality
- Testing with real Google authentication (Firebase Auth)
- Testing features that require logged-in state
- Debugging extension interactions

**Prerequisites:**
1. Close all Chrome windows
2. Start Chrome with remote debugging:

```bash
# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/Library/Application Support/Google/Chrome"

# Linux
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.config/google-chrome"

# Windows
"C:\Program Files\Google\Chrome\Application\chrome.exe" ^
  --remote-debugging-port=9222 ^
  --user-data-dir="%LOCALAPPDATA%\Google\Chrome\User Data"
```

3. Log into Google/Firebase in Chrome if needed
4. Install/enable your extension if testing extension

**Configuration:**
```typescript
// playwright.config.ts or test file
import { chromium } from 'playwright';

const browser = await chromium.connectOverCDP('http://localhost:9222');
const context = browser.contexts()[0];  // Use existing context
const page = context.pages()[0] || await context.newPage();
```

**Run:**
```bash
# Custom script for profile mode
npx playwright test --config=playwright.profile.config.ts
```

## Instructions

### 1. Determine Mode

Check if test requires authentication or extension:

```bash
# Check test file for extension or auth requirements
grep -l "extension\|chrome\|auth\|login\|firebase" frontend/e2e/*.spec.ts
```

If matches found or user specified `--extension`, use Google Profile mode.

### 2. Interactive Mode Selection (if gum available)

```bash
if command -v gum &> /dev/null; then
  MODE=$(gum choose --header "🎭 Select Playwright mode:" \
    "Chromium (isolated)" \
    "Google Profile (auth + extensions)")

  case "$MODE" in
    "Chromium"*) USE_PROFILE=false ;;
    "Google"*)   USE_PROFILE=true ;;
  esac
else
  # Default to chromium unless --extension specified
  USE_PROFILE=false
fi
```

### 3. Setup for Google Profile Mode

If using Google Profile mode:

```bash
echo "📍 Google Profile Mode Setup"
echo ""
echo "1. Close all Chrome windows"
echo "2. Run this command to start Chrome with debugging:"
echo ""
gum style --border rounded --padding "1 2" \
  "google-chrome --remote-debugging-port=9222 --user-data-dir=\"\$HOME/.config/google-chrome\""
echo ""
echo "3. Wait for Chrome to open, then log in if needed"
echo ""

if command -v gum &> /dev/null; then
  gum confirm "Chrome is running with debugging port 9222?" || exit 1
else
  read -p "Press Enter when Chrome is ready..."
fi

# Verify connection
curl -s http://localhost:9222/json/version > /dev/null || {
  echo "❌ Cannot connect to Chrome on port 9222"
  echo "   Make sure Chrome is running with --remote-debugging-port=9222"
  exit 1
}
echo "✅ Connected to Chrome"
```

### 4. Run Tests

```bash
cd frontend

if [ "$USE_PROFILE" = true ]; then
  # Google Profile mode
  echo "🎭 Running with Google Profile..."
  PLAYWRIGHT_CHROMIUM_USE_CDP=true \
  CDP_ENDPOINT=http://localhost:9222 \
  npx playwright test $TEST_PATTERN --reporter=list
else
  # Standard Chromium mode
  echo "🎭 Running with attached Chromium..."
  npx playwright test $TEST_PATTERN --reporter=list
fi
```

### 5. Handle Failures

If tests fail, offer options:

```bash
if [ $? -ne 0 ]; then
  echo ""
  echo "❌ Some tests failed"

  if command -v gum &> /dev/null; then
    ACTION=$(gum choose --header "What next?" \
      "View report (playwright show-report)" \
      "Debug failing test (--debug)" \
      "Take screenshot and continue" \
      "Exit")

    case "$ACTION" in
      "View"*)   npx playwright show-report ;;
      "Debug"*)  npx playwright test --debug $FAILED_TEST ;;
      "Take"*)   # Use MCP to take screenshot ;;
      "Exit")    exit 1 ;;
    esac
  fi
fi
```

## Output Format

```
═══════════════════════════════════════════════════════════════
  🎭 E2E TESTING
═══════════════════════════════════════════════════════════════

  Mode: {Chromium | Google Profile}
  Tests: {pattern or "all"}

  Running...

  ✅ auth.spec.ts (3 tests)
  ✅ navigation.spec.ts (5 tests)
  ❌ extension.spec.ts (1/2 failed)
     └─ "should capture transcript" - timeout

═══════════════════════════════════════════════════════════════
  Results: 9/10 passed

  ➡️  Next steps:
     npx playwright show-report  ← View detailed report
     /testing:e2e extension      ← Re-run failed tests

═══════════════════════════════════════════════════════════════
```

## Quick Reference

| Scenario | Mode | Command |
|----------|------|---------|
| Basic UI tests | Chromium | `/testing:e2e` |
| Auth flow tests | Profile | `/testing:e2e auth --mode=profile` |
| Extension tests | Profile | `/testing:e2e --extension` |
| CI/CD | Chromium (headless) | `npx playwright test` |
| Debug specific | Either | `npx playwright test --debug test.spec.ts` |

## Troubleshooting

### "Cannot connect to Chrome on port 9222"
- Close ALL Chrome windows first
- Start Chrome with the exact command shown above
- Check: `curl http://localhost:9222/json/version`

### Extension not visible
- Ensure extension is installed in the profile
- Check `chrome://extensions` - must be enabled
- Profile mode must use same user-data-dir as normal Chrome

### "No contexts available"
- Open at least one tab in Chrome before running tests
- Playwright needs an existing context to attach to
