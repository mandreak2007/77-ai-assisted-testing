# Playwright CLI Command Reference

Complete reference for running, debugging, reporting, and maintaining Playwright tests via the CLI.

---

## Running Tests

### Basic Run Commands

```bash
# Run all tests
npx playwright test

# Run a specific file
npx playwright test tests/authentication/login.spec.ts

# Run all specs in a folder
npx playwright test tests/authentication/

# Run multiple specific files
npx playwright test tests/authentication/login.spec.ts tests/checkout/checkout.spec.ts
```

---

### Filtering Tests

```bash
# Filter by test name pattern (substring or regex)
npx playwright test --grep "successful login"
npx playwright test --grep "@smoke"
npx playwright test --grep "@regression"

# Invert filter — run everything EXCEPT matching
npx playwright test --grep-invert "@wip"

# Run tests matching a tag (Playwright v1.42+ tag syntax)
npx playwright test --grep "@smoke"
```

---

### Browser Projects

```bash
# Run only in Chromium
npx playwright test --project=chromium

# Run only in Firefox
npx playwright test --project=firefox

# Run only in WebKit
npx playwright test --project=webkit

# Run in mobile Chrome
npx playwright test --project="mobile-chrome"

# Run in multiple specific projects
npx playwright test --project=chromium --project=firefox
```

---

### Execution Options

```bash
# Run in headed mode (see the browser)
npx playwright test --headed

# Run with a specific number of workers
npx playwright test --workers=4

# Run serially (1 worker, useful for debugging)
npx playwright test --workers=1

# Disable parallelism entirely
npx playwright test --fully-parallel=false

# Retry failed tests N times
npx playwright test --retries=2

# Repeat each test N times (flakiness detection)
npx playwright test --repeat-each=3

# Run only the tests that failed in the last run
npx playwright test --last-failed

# Set timeout (ms) for each test
npx playwright test --timeout=60000
```

---

## Debugging Tests

```bash
# Debug mode — opens Playwright Inspector, pauses on each action
npx playwright test --debug

# Debug a specific file
npx playwright test tests/authentication/login.spec.ts --debug

# Debug a specific test by name
npx playwright test --debug --grep "successful login"

# Run in UI mode — interactive test explorer and time-travel debugging
npx playwright test --ui

# Run with slow-motion (ms delay between actions)
npx playwright test --headed --slowmo=500

# Enable verbose output
npx playwright test --reporter=list

# Pause at a specific point in the test (add to the spec):
# await page.pause();
```

---

## Tracing

```bash
# Enable trace for all tests (for debugging)
npx playwright test --trace=on

# Enable trace only on first retry (default CI config)
npx playwright test --trace=on-first-retry

# View a trace file
npx playwright show-trace test-results/test-trace.zip

# Or open the Playwright Trace Viewer online
# Upload the zip at: https://trace.playwright.dev
```

---

## Reporting

```bash
# Show the HTML report (opens in browser)
npx playwright show-report

# Show a specific report folder
npx playwright show-report reports/playwright-html

# Generate report in specific format
npx playwright test --reporter=html
npx playwright test --reporter=json
npx playwright test --reporter=list
npx playwright test --reporter=dot
npx playwright test --reporter=junit

# Multiple reporters (also configurable in playwright.config.ts)
npx playwright test --reporter=list,html
```

---

## Code Generation

```bash
# Open codegen — browser opens and records your actions as Playwright code
npx playwright codegen

# Codegen for a specific URL
npx playwright codegen http://localhost:3000/login

# Codegen with a specific browser
npx playwright codegen --browser=firefox http://localhost:3000

# Codegen with a specific viewport
npx playwright codegen --viewport-size=1280,720 http://localhost:3000

# Codegen and save output to a file
npx playwright codegen --output=tests/generated.spec.ts http://localhost:3000

# Codegen using a saved storage state (authenticated session)
npx playwright codegen --load-storage=test-data/user-state.json http://localhost:3000
```

---

## Screenshots and Screenshots-on-Failure

```bash
# Take a screenshot manually during a run (add to spec file):
# await page.screenshot({ path: 'reports/screenshots/my-page.png', fullPage: true });

# Screenshots are automatically captured on failure when configured:
# use: { screenshot: 'only-on-failure' }   ← in playwright.config.ts

# Video is retained on failure when configured:
# use: { video: 'retain-on-failure' }      ← in playwright.config.ts
```

---

## TypeScript Validation

```bash
# Type-check all files without running tests
npx tsc --noEmit

# Watch mode for type checking during development
npx tsc --noEmit --watch
```

---

## Installing Browsers

```bash
# Install all browsers
npx playwright install

# Install a specific browser
npx playwright install chromium
npx playwright install firefox
npx playwright install webkit

# Install with OS dependencies (required on Linux CI)
npx playwright install --with-deps

# Install a specific browser with deps
npx playwright install --with-deps chromium
```

---

## Useful package.json Scripts

Add these to `package.json` for convenience:

```json
{
  "scripts": {
    "test":            "playwright test",
    "test:smoke":      "playwright test --grep @smoke",
    "test:regression": "playwright test --grep @regression",
    "test:headed":     "playwright test --headed",
    "test:ui":         "playwright test --ui",
    "test:debug":      "playwright test --debug",
    "test:chromium":   "playwright test --project=chromium",
    "test:firefox":    "playwright test --project=firefox",
    "test:last-failed":"playwright test --last-failed",
    "codegen":         "playwright codegen",
    "report":          "playwright show-report",
    "typecheck":       "tsc --noEmit"
  }
}
```

---

## Environment Variables for CLI

```bash
# Override BASE_URL at run time
BASE_URL=https://staging.example.com npx playwright test

# Run headlessly in CI (Playwright is headless by default, this is explicit)
HEADLESS=true npx playwright test

# Pass credentials at run time (for CI secrets)
TEST_USER_EMAIL=ci@example.com TEST_USER_PASSWORD=CIPass123 npx playwright test
```

---

## CI Quick Reference

```bash
# Standard CI pipeline sequence
npm ci
npx playwright install --with-deps chromium
npx playwright test --project=chromium --grep @smoke
npx playwright test --project=chromium
npx playwright show-report  # only locally; in CI, upload the artifact
```
