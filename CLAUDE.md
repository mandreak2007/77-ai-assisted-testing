# CLAUDE.md — Playwright CLI (BDD) Automation Project

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This file defines Claude's behavior for this project. Claude reads it at the start of every session and follows all instructions exactly.

---

## Project Overview

This project uses a **single automation mode**: **Playwright CLI (BDD)** with `playwright-bdd`.

| Aspect | Value |
|---|---|
| **Runner** | `npx bddgen && npx playwright test` |
| **BDD Layer** | [`playwright-bdd`](https://vitalets.github.io/playwright-bdd/) — Gherkin `.feature` files compiled into native Playwright tests |
| **Gherkin Parser** | `@cucumber/gherkin` + `@cucumber/messages` (installed by `playwright-bdd`) |
| **Test Files** | `.feature` files in `features/` + step definitions in `step-definitions/` |
| **Page Objects** | `src/pages/` with `SelfHealingLocator` |
| **Fixtures** | `src/fixtures/test-fixtures.ts` — injected into step definitions via `createBdd(test)` |
| **Reports** | Playwright HTML report + Allure (`allure-playwright`) |

The project shares: self-healing locators, Page Object Model, `SelfHealingLocator`, `LocatorRegistry`, requirements analysis via `qa-test-analyst`, and CI/CD via GitHub Actions.

> **Note:** MCP (Model Context Protocol) mode is NOT used in this project. Do not scaffold `.mcp.json`, `cucumber.js`, `@playwright/mcp`, or `@cucumber/cucumber` runner packages. All BDD execution goes through `playwright-bdd` → `bddgen` → `npx playwright test`.

---

## Master Workflow

Claude follows these phases in strict order.

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1 — SETUP                                                │
│  Detect / scaffold the CLI (BDD) project structure, install     │
│  all Gherkin + BDD dependencies, and generate the                │
│  PROJECT_OVERVIEW.pdf documentation (with full CLI reference).  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ structure confirmed
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2 — REQUIREMENTS ANALYSIS (conversational)               │
│  User drives the conversation: summaries, gap analysis, TC      │
│  planning, duplicate checks. The qa-test-analyst skill is       │
│  invoked ONLY when the user explicitly asks to "create gherkin  │
│  tests" / "generate the feature file" / "update the RTM".       │
└──────────────────────────────┬──────────────────────────────────┘
                               │ user asks to proceed
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3 — EXPLORATION (3A, optional)                            │
│  Browse the live app for a chosen scenario. Capture locators,   │
│  observed data, network + console evidence. Ask the user any    │
│  clarifying questions before scripting.                         │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 3 — SCRIPT GENERATION (3B, optional)                     │
│  Automate ONLY the specific test case(s) the user names.        │
│  Add @ready tag to each scripted scenario. Recommend the        │
│  next-easiest test case to script next (highest reuse).         │
└──────────────────────────────┬──────────────────────────────────┘
                               │ scripts ready
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4 — EXECUTION WITH SELF-HEALING                          │
│  npx bddgen && npx playwright test → self-healing loop.         │
└─────────────────────────────────────────────────────────────────┘
```

---

## PHASE 1 — Project Setup

**Trigger:** Start of every session, automatically, before anything else.

**Skill:** `playwright-cli-automation`

### 1A — Scan for Existing CLI Project

```bash
# PowerShell (Windows)
Get-ChildItem -Recurse -Depth 3 -Filter "playwright.config.ts" -ErrorAction SilentlyContinue
Get-ChildItem -Recurse -Filter "*.feature" -ErrorAction SilentlyContinue | Select-Object -First 10
Get-ChildItem -Path "step-definitions" -Recurse -Filter "*.ts" -ErrorAction SilentlyContinue
Get-ChildItem -Path "src/pages" -Filter "*.ts" -ErrorAction SilentlyContinue
Get-ChildItem -Path "src/fixtures" -Filter "*.ts" -ErrorAction SilentlyContinue
```

Confirm the project is a `playwright-bdd` project by grepping `playwright.config.ts` for `defineBddConfig`.

**If project exists** — summarize and stop:

```
✅ Playwright CLI (BDD) project detected.

📁 Project summary
   Config:       playwright.config.ts  ✓  (playwright-bdd)
   Features:     features/  →  X folders | X files | X scenarios
   Steps:        step-definitions/  →  X files
   Pages:        src/pages/  →  X files
   Fixtures:     src/fixtures/test-fixtures.ts  ✓
   Requirements: requirements/  →  X files

Ready. What would you like to do?
```

**If no project exists** — scaffold immediately (no confirmation needed). Load `references/scaffold-templates.md` from the `playwright-cli-automation` skill and create every file listed below:

```
playwright.config.ts            ← playwright-bdd config with defineBddConfig
global-setup.ts                 ← reachability check before suite
global-teardown.ts              ← healing summary after suite
tsconfig.json
package.json                    ← @playwright/test + playwright-bdd (NO MCP, NO @cucumber/cucumber runner)
.env.example
.gitignore
src/
  pages/
    BasePage.ts
  fixtures/
    test-fixtures.ts            ← Playwright fixtures; used by createBdd(test) in step defs
  utils/
    SelfHealingLocator.ts
    LocatorRegistry.ts
    ApiHelper.ts
features/                       ← Gherkin .feature files (executed by playwright-bdd)
  <module>/
    <module>.feature
step-definitions/               ← TypeScript step bindings (playwright-bdd)
  <module>/
    <module>.steps.ts
.features-gen/                  ← auto-generated by bddgen (do not edit)
test-data/
  users.json
requirements/                   ← place BRD / story files here
documentation/
  PROJECT_OVERVIEW.pdf           ← auto-generated project documentation
reports/
  locator-registry.json
.github/
  workflows/
    playwright.yml
```

The `requirements/` folder must be created as part of scaffolding. Do not skip it even if empty.

After scaffolding all files, **automatically run ALL of the following steps in order** (do not ask the user — just run them).

---

### Step 1 — Install Gherkin + BDD Dependencies

Every fresh scaffold and every existing project **without** a `node_modules/playwright-bdd` folder must run the full install below.

**1.1 — Install runtime dev dependencies**

```bash
# Core Playwright test runner
npm install --save-dev @playwright/test@^1.44.0

# BDD / Gherkin layer (compiles .feature → native Playwright tests)
npm install --save-dev playwright-bdd@^7.0.0

# Gherkin parser + Cucumber expressions (peer deps of playwright-bdd)
npm install --save-dev @cucumber/gherkin@^28.0.0
npm install --save-dev @cucumber/messages@^24.0.0
npm install --save-dev @cucumber/cucumber-expressions@^17.0.0

# TypeScript toolchain
npm install --save-dev typescript@^5.0.0 ts-node@^10.9.0 tsconfig-paths@^4.2.0 @types/node@^20.0.0

# Config / utilities
npm install --save-dev dotenv@^16.0.0

# Reporting
npm install --save-dev allure-playwright@^2.13.0 multiple-cucumber-html-reporter@^3.0.0

# Linting (optional but recommended)
npm install --save-dev eslint@^8.0.0 @typescript-eslint/eslint-plugin@^7.0.0 @typescript-eslint/parser@^7.0.0
```

**1.2 — Install Playwright browsers**

```bash
npx playwright install --with-deps
```

**1.3 — Verify the BDD stack is wired**

Run these three verifications and abort with a clear error if any fails:

```bash
# playwright-bdd CLI must be discoverable
npx bddgen --help

# Playwright runner must be discoverable
npx playwright --version

# TypeScript must compile cleanly
npx tsc --noEmit
```

**1.4 — Confirm `package.json` scripts are BDD-CLI-aligned**

The generated `package.json` **must** include exactly these scripts (drop any Cucumber-CLI or MCP-related scripts from prior templates):

```json
{
  "scripts": {
    "bdd:gen": "bddgen",
    "test": "bddgen && playwright test",
    "test:headed": "bddgen && playwright test --headed",
    "test:ui": "bddgen && playwright test --ui",
    "test:debug": "bddgen && playwright test --debug",
    "test:smoke": "bddgen && playwright test --grep @smoke",
    "test:regression": "bddgen && playwright test --grep @regression",
    "test:chromium": "bddgen && playwright test --project=chromium",
    "test:firefox": "bddgen && playwright test --project=firefox",
    "test:webkit": "bddgen && playwright test --project=webkit",
    "report:html": "playwright show-report",
    "report:allure": "allure generate reports/allure-results --clean -o reports/allure-report && allure open reports/allure-report",
    "lint": "eslint src/**/*.ts step-definitions/**/*.ts",
    "typecheck": "tsc --noEmit",
    "healing:report": "node scripts/print-healing-report.js",
    "healing:clear": "node -e \"require('fs').rmSync('reports/locator-registry.json',{force:true})\""
  }
}
```

**1.5 — Forbidden packages (auto-remove if present)**

If the following packages exist in `package.json`, remove them immediately with `npm uninstall`:

| Package | Reason |
|---|---|
| `@playwright/mcp` | MCP mode not used in this project |
| `@cucumber/cucumber` | Replaced by `playwright-bdd` — the classic Cucumber runner is not used |
| `allure-cucumberjs` | Uses classic Cucumber runner — replaced by `allure-playwright` |

**1.6 — Configure `playwright.config.ts` for BDD**

The config **must** call `defineBddConfig` from `playwright-bdd`:

```typescript
import { defineConfig, devices } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';
import * as dotenv from 'dotenv';

dotenv.config();

const testDir = defineBddConfig({
  features: 'features/**/*.feature',
  steps: 'step-definitions/**/*.steps.ts',
  outputDir: '.features-gen',
});

export default defineConfig({
  testDir,
  timeout: 60_000,
  expect: { timeout: 10_000 },
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  reporter: [
    ['list'],
    ['html', { outputFolder: 'reports/playwright-html', open: 'never' }],
    ['allure-playwright', { outputFolder: 'reports/allure-results' }],
  ],
  use: {
    baseURL: process.env.BASE_URL,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit',   use: { ...devices['Desktop Safari'] } },
  ],
  globalSetup: './global-setup.ts',
  globalTeardown: './global-teardown.ts',
});
```

---

### Step 2 — Install PDF Generation Tools

Check whether `pandoc` and `wkhtmltopdf` are already installed:

```bash
pandoc --version
wkhtmltopdf --version
```

For any tool that is missing, run the install command that matches the current OS:

| OS | Tool | Install command |
|---|---|---|
| **Windows** | pandoc | `winget install --id JohnMacFarlane.Pandoc -e --silent` |
| **Windows** | wkhtmltopdf | `winget install --id wkhtmltopdf.wkhtmltopdf -e --silent` |
| **macOS** | pandoc | `brew install pandoc` |
| **macOS** | wkhtmltopdf | `brew install --cask wkhtmltopdf` |
| **Linux (apt)** | pandoc | `sudo apt-get install -y pandoc` |
| **Linux (apt)** | wkhtmltopdf | `sudo apt-get install -y wkhtmltopdf` |

If `winget` is unavailable on Windows, fall back to Chocolatey:
```bash
choco install pandoc -y
choco install wkhtmltopdf -y
```

---

### Step 3 — Generate PDF Documentation

Create `documentation/PROJECT_OVERVIEW.pdf` from `references/project-overview-template.md` in the `playwright-cli-automation` skill:

1. Write the template content to `documentation/_tmp_overview.md`.
2. Run: `pandoc documentation/_tmp_overview.md -o documentation/PROJECT_OVERVIEW.pdf --pdf-engine=wkhtmltopdf`
3. Delete `documentation/_tmp_overview.md`.

If PDF generation fails despite the tools being installed, print the exact error output, leave `_tmp_overview.md` in place, and report the failure clearly to the user.

---

### Setup Completion Message

Once all steps complete, confirm to the user:

```
✅ Playwright CLI (BDD) project scaffolded and dependencies installed.

   BDD stack        : playwright-bdd + @cucumber/gherkin + @cucumber/cucumber-expressions ✓
   Playwright core  : @playwright/test + browsers ✓
   Requirements dir : requirements/ ✓  (drop your BRD or user story here)
   Documentation    : documentation/PROJECT_OVERVIEW.pdf ✓  (includes full CLI command reference)

Common CLI commands you can use at any time:
  npm test                                      — run all
  npx bddgen && npx playwright test --grep "@ready"    — run only scripted scenarios
  npm run test:ui                               — interactive UI mode
  npm run test:headed                           — see the browser
  npm run test:smoke                            — @smoke scenarios only
  npm run test:regression                       — @regression scenarios only
  npm run report:html                           — open the HTML report

Full CLI reference is in documentation/PROJECT_OVERVIEW.pdf.

What would you like to do next?
  • Drop a requirements file in requirements/ and ask me to walk through it
  • Ask me to create gherkin tests from an existing file
  • Ask me to explore a specific flow on the live app
  • Ask me to automate a specific test case
```

**After setup: STOP. Wait for user input before proceeding.**

---

## PHASE 2 — Requirements Analysis (Conversational)

**Trigger:** User places a file in `requirements/`, pastes requirements content, or asks any analysis-related question.

Phase 2 is **conversational and user-led**. The user decides the depth and shape of the work. Do not jump straight into feature-file generation.

### 2.1 — Default Mode: Talk First, Build When Asked

When the user drops a requirements file or brings up requirements, **read the input, understand it, then engage the user**. Ask what they want next — do not assume they want Gherkin scenarios.

| The user might want… | Respond by… |
|---|---|
| A plain-language walk-through of the requirement | Summarise the requirement, its actors, business rules, and edge cases in prose |
| A gap analysis or requirement critique | Point out ambiguities, missing constraints, contradictions — no files written |
| A test coverage plan (positive / negative / boundary breakdown) | Draft the test plan as a chat table; no files written |
| A list of TC titles + intents | Print candidate test-case titles for review; no files written |
| Duplicate detection against the existing master RTM | Compare against `reports/master-rtm.xlsx`; report matches |
| A Gherkin `.feature` file + RTM row generation | Explicit trigger required — see 2.3 below |
| Discussion, brainstorming, "what would you test?" | Chat freely, offer ideas, ask questions |

Suggested opening after reading a requirements file:

```
📄 Read: <filename> — <one-line summary>

I can help with any of these — what would you like?
  1. Summarise / explain the requirement
  2. Do a gap analysis
  3. Draft a test-coverage plan (no files written)
  4. List candidate test-case titles
  5. Check for duplicates against the existing RTM
  6. Generate Gherkin scenarios + update the master RTM   ← invokes qa-test-analyst
  7. Something else — tell me what you need
```

### 2.2 — Talk Naturally

- The user may iterate — "revise scenario 3", "add a boundary case for max length", "what about accessibility?" — respond in-line without invoking any skill.
- Keep notes of assumptions and open questions in the conversation; only commit them to a file when the user asks for the file.
- If the user pushes back on scope ("just three critical scenarios, not the whole thing"), honour it. Do not exhaustively expand coverage unless requested.

### 2.3 — Gherkin Test Case Generation (Skill Trigger)

**The `qa-test-analyst` skill is invoked only** when the user explicitly asks for Gherkin/BDD test cases or the RTM to be produced. Trigger phrases:

- "create gherkin tests"
- "generate gherkin scenarios"
- "write test cases" / "create test cases"
- "create a feature file" / "generate the .feature"
- "generate the RTM" / "update the RTM"
- "build the traceability matrix"
- Explicit selection of option 6 in the menu above

Once triggered, run the skill's full pipeline: existing-RTM load → duplicate detection → coverage planning → Gherkin authoring → RTM update → three-pass quality review → delivery summary. The full procedure lives in the skill; do not repeat it here.

**Do not trigger the skill implicitly.** Reading a BRD, discussing coverage, or listing candidate TC titles is *not* a trigger. Wait for the explicit ask.

### 2.4 — After the Skill Runs

The `qa-test-analyst` skill prints its own Delivery Summary. After it finishes:

- Confirm which files were created / updated.
- Ask the user what to do next (explore the app, generate scripts for a specific scenario, or continue analysis).
- Do **not** jump into Phase 3 automatically.

---

## PHASE 3 — App Exploration & Automation Script Generation

**Skill:** `playwright-cli-automation`

Phase 3 has **two optional, independent parts**. Either can be run without the other, and both are user-driven:

- **3A — App Exploration** (optional): The framework navigates the live application for a specific scenario, captures locators and test data as evidence, and asks the user any clarifying questions before scripting begins.
- **3B — Script Generation** (optional): The framework automates **only** the specific test case(s) the user names, marks them `@ready` in the feature file, and recommends the next-closest test case that would take the least additional effort.

Do not run 3A automatically before 3B unless the user asks. If the user goes straight to "automate TC-XXX", proceed directly to 3B — Exploration is an aid, not a gate.

---

### PART 3A — App Exploration (Live Discovery)

**Trigger:** "explore the app", "walk through the login flow", "capture the locators for X", "browse and record", "traverse the checkout page", "map the DOM for TC-XXX", or similar.

Use the Playwright MCP browser tools (`mcp__playwright__browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type`, `browser_evaluate`, `browser_take_screenshot`, `browser_console_messages`, `browser_network_requests`) to walk through the requested scenario against the running application.

#### 3A.1 — Confirm the Scenario Scope

Resolve which scenario to explore. If unclear, ask:

- Which page or flow (e.g. "the login screen", "checkout step 2", TC-056)?
- Which URL / environment (`BASE_URL` or an explicit URL)?
- Any prerequisites (existing logged-in user, seeded data, feature flag)?

Do **not** start browsing until the scope is clear.

#### 3A.2 — Walk the Flow & Capture Assets

For each step in the scenario:

1. Navigate to the entry URL with `browser_navigate`.
2. Take an accessibility snapshot with `browser_snapshot` — this is the source of truth for locators.
3. For every element the scenario interacts with (input, button, link, alert, table row, modal), collect **all six fallback candidates** in priority order:
   1. `data-testid` (or `data-test`, `data-qa`)
   2. `aria-label` / role + accessible name
   3. `placeholder`
   4. Visible text / label text
   5. CSS selector (class or id)
   6. XPath
4. Record observed test data (real placeholder values, min/max lengths, required-field indicators, dropdown options, validation messages, URL patterns after actions).
5. Capture network calls (`browser_network_requests`) and console errors (`browser_console_messages`) that fire during the flow — useful for later assertions and back-end wiring.
6. Take a screenshot per meaningful state with `browser_take_screenshot` and save under `reports/exploration/<scenario-slug>/`.

#### 3A.3 — Persist the Evidence

Write a single exploration report to `reports/exploration/<scenario-slug>.md` containing:

- Scenario name and TC-ID (if provided)
- Starting URL and any preconditions
- Ordered list of steps taken
- Per-element locator table (all six strategies + which one was actually observed)
- Observed validation messages, error text, and URL transitions
- Any network endpoints called
- Screenshots referenced by path
- Open questions for the user (see 3A.4)

This report becomes the input for 3B if the user proceeds to scripting.

#### 3A.4 — Ask Clarifying Questions

After the walk, print a short question list — **only** ask what genuinely blocks scripting:

```
🔎 Exploration complete for: <scenario>

I captured N elements and M state transitions. Before I write the script,
please confirm:

  1. <question — e.g. "The 'Remember me' checkbox is optional. Should the test
     verify both checked and unchecked states, or only the default (unchecked)?">
  2. <question — e.g. "After login the app redirects to /dashboard OR /home
     depending on user role. Which role should the test use?">
  3. <question — e.g. "Validation error 'Email is required' takes ~800ms to appear.
     Is that expected, or should the test flag it as a performance issue?">

Skip any that don't apply. Once answered, I can proceed to generate the script (3B).
```

If there are no blocking questions, say so explicitly and offer to proceed straight to 3B.

#### 3A.5 — Exploration Report Summary

```
🔎 App Exploration — <scenario>

   URL walked        : <url>
   Steps captured    : XX
   Elements captured : XX  (all with 6-strategy fallback candidates)
   Screenshots       : reports/exploration/<slug>/*.png
   Evidence file     : reports/exploration/<slug>.md

Ready for scripting when you are. Say "generate script for <TC-XXX>".
```

**After 3A: STOP.** Wait for the user to answer clarifying questions or ask for 3B.

---

### PART 3B — Automation Script Generation (Test-Case-Scoped)

**Trigger:** Explicit request naming a scenario, TC-ID, or feature file. Examples:

- "generate the script for TC-056"
- "automate the successful login scenario"
- "script `features/authentication/login.feature`"
- "automate this" (only when the current conversation makes the scope unambiguous)

**Do NOT auto-scope to "everything".** If the user says "automate everything", **confirm the exact scope back** and preferably narrow to the smallest meaningful batch (one feature file, one TC-ID range).

#### 3B.1 — Resolve Scope Strictly

| Prompt | Resolved scope |
|---|---|
| "generate script for TC-056" | Only that one scenario's steps |
| "automate `login.feature`" | All scenarios in that single feature file |
| "automate all @smoke in authentication/" | Scenarios matching the tag filter, in that folder only |
| "automate everything" | Confirm back before proceeding — never assume |

If scope is ambiguous, list candidates and ask before writing any file. Never generate more than the user asked for.

#### 3B.2 — Load Exploration Evidence (if present)

If `reports/exploration/<scenario-slug>.md` exists for a scenario in scope, load it and use its locator table + observed data as the primary source when building the Page Object. Otherwise, fall back to the feature file alone.

#### 3B.3 — Parse Feature File(s) in Scope

Extract only the scenarios in scope:
- **Feature name** → Page Object class name and step definition file name
- **Steps in scope** → POM methods and step definition bindings needed
- **Background** → shared `Given` steps (reuse if already defined elsewhere)
- **Scenario Outline + Examples** → data-driven steps with `{string}` / `{int}` Cucumber expressions
- **Tags** → passed through by `bddgen`; used at runtime with `--grep`

#### 3B.4 — Generate / Extend the Page Object

Save to: `src/pages/<FeatureName>Page.ts`. Load `references/self-healing.md` from the `playwright-cli-automation` skill for the `SelfHealingLocator` / `LocatorRegistry` pattern.

Rules:
- Every class extends `BasePage`.
- Every locator uses `SelfHealingLocator` with all six fallback strategies (from exploration evidence if available).
- Locator priority order: `data-testid` → `aria-label`/role → placeholder → visible text → CSS → XPath.
- All locators are `private readonly` in the constructor.
- All public methods are `async` returning `Promise<void>` or a typed value.
- Method names reflect business intent (`loginWith()`, `expectRedirectedToDashboard()`).
- No assertions inside Page Objects — assertions belong in step definitions only.
- If the Page Object already exists, append only the methods the in-scope scenarios need — never overwrite existing ones.

#### 3B.5 — Generate / Extend Step Definitions

Save to: `step-definitions/<module>/<module>.steps.ts`. Load `references/code-patterns.md` from the `playwright-cli-automation` skill.

Rules:
- `import { createBdd } from 'playwright-bdd'` and the custom `test` from `src/fixtures/test-fixtures`.
- `const { Given, When, Then } = createBdd(test)` — **never** import from `@cucumber/cucumber`.
- No locators or page logic inside step definitions — delegate to POM methods.
- Fixtures injected as the first parameter (no `CustomWorld`, no `this`).
- All steps `async` with proper `await`.
- Cucumber expressions (`{string}`, `{int}`, `{float}`) for `Scenario Outline` parameters.
- Scan every existing `*.steps.ts` first — never duplicate a step pattern; reuse or extend.

Example:
```typescript
// step-definitions/authentication/login.steps.ts
import { createBdd } from 'playwright-bdd';
import { test } from '../../src/fixtures/test-fixtures';

const { Given, When, Then } = createBdd(test);

Given('the user is on the login page', async ({ loginPage }) => {
  await loginPage.goto();
});

When('the user logs in with valid credentials', async ({ loginPage }) => {
  await loginPage.loginWith(
    process.env.TEST_USER_EMAIL!,
    process.env.TEST_USER_PASSWORD!
  );
});

Then('the user should be redirected to the dashboard', async ({ loginPage }) => {
  await loginPage.expectRedirectedToDashboard();
});
```

#### 3B.6 — Register Fixtures

For each new Page Object, update `src/fixtures/test-fixtures.ts`:
- Add the import
- Add the fixture property to the `TestFixtures` type
- Add the fixture initializer inside `test.extend()`

The same `test` object is passed to `createBdd(test)` in every step definition file — this is how fixtures reach steps without `CustomWorld`.

#### 3B.7 — Tag the Scripted Scenario(s) as `@ready`

`@ready` is a **scenario-level** tag applied only after the scenario is fully wired.

Rules:
- Add `@ready` as the **first tag** on the scenario's tag line, immediately after its steps are generated and the corresponding step definitions verified to exist.
- Tag each scenario individually — never bulk-tag scenarios that were not scripted this run.
- If `@ready` is already present on a scenario, skip — do not duplicate.
- A scenario is `@ready` **only when** every one of its steps has a matching step-definition binding in the corresponding `*.steps.ts` file.
- Also add `@ready` on the line immediately before `Feature:` **only when every scenario** in the feature file is `@ready`.

Example — before scripting:
```gherkin
  @tc-TC-056 @req-REQ-026 @positive @smoke
  Scenario: TC-056 — Login page renders all required form controls
```

Example — after scripting:
```gherkin
  @ready @tc-TC-056 @req-REQ-026 @positive @smoke
  Scenario: TC-056 — Login page renders all required form controls
```

#### 3B.8 — Recommend the Next Test Case (Effort-Aware)

After generation completes, scan the remaining un-`@ready` scenarios in the same feature file (and secondary: the same domain folder) and recommend the **one** that would take the least incremental effort to script next.

Effort estimation heuristic — a scenario is "cheap" when it maximises reuse of what was just built:

| Signal | Weight |
|---|---|
| Uses the same Page Object just created / extended | +3 |
| Reuses ≥50% of the Given/When/Then step patterns just written | +3 |
| Same URL entry point | +1 |
| Same actor / precondition | +1 |
| Adds ≤2 net-new locators | +2 |
| Adds ≤2 net-new step patterns | +2 |
| No new Page Object required | +2 |
| Already covered by exploration evidence | +2 |

Rank candidates by total score and recommend the top one:

```
💡 Next-easiest scenario to automate:

   TC-057 — Login rejects unregistered email
     Reuses         : LoginPage (already created) — same POM
                     3 of 4 steps already written (only 1 new "Then")
     New work needed: 1 new step definition, 1 new POM method
                     (expectEmailNotRegisteredError)
     Estimated cost : ~10 minutes (~85% reuse)

   Want me to generate it? (yes / no / pick a different TC)
```

Only ever recommend **one** next case per generation report. Never queue a batch on your own.

#### 3B.9 — Generation Report

```
✅ Scripts generated for: <scope>

   Scenario(s) scripted:
     • TC-056 — Login page renders all required form controls           @ready ✓

   Files touched:
     src/pages/LoginPage.ts                              ✓  (X methods added)
     src/fixtures/test-fixtures.ts                       ✓  (loginPage fixture added)
     step-definitions/authentication/login.steps.ts      ✓  (X steps added)
     features/authentication/login.feature               ✓  (@ready tag added to TC-056)

   Scenarios in scope   : 1 / 1  scripted
   Scenarios in feature : 1 / 12 marked @ready

Ready to run:
  npm test                                              — full suite
  npx bddgen && npx playwright test --grep "@ready"     — only scripted scenarios
  npx bddgen && npx playwright test --grep "TC-056"     — this scenario only

💡 Next-easiest scenario: <see 3B.8 recommendation above>
```

---

## PHASE 4 — Execution with Self-Healing

**Trigger:** Scripts generated. User says "run tests", "execute", or confirms after Phase 3.

**Skill:** `playwright-cli-automation`

### 4A — Run Tests

```bash
# Generate test files from features (always run before executing)
npx bddgen

# Run all tests
npm test                                      # → bddgen && playwright test

# Run by tag
npm run test:smoke                            # → --grep @smoke
npm run test:regression                       # → --grep @regression

# Run a specific scenario by name pattern
npx bddgen && npx playwright test --grep "successful login"

# Run in a specific browser project
npm run test:chromium
npm run test:firefox
npm run test:webkit

# Run headed / UI / debug
npm run test:headed
npm run test:ui
npm run test:debug

# View the HTML report after a run
npm run report:html                           # → playwright show-report
npm run report:allure                         # → allure generate + open
```

### 4B — Self-Healing Iteration Loop

After every run, for each failing test:

```
1. Read the error type:

   a) SelfHealingLocator.NoStrategyResolvedError
      → Add new fallback strategies to the Page Object.
      → Use --debug or the HTML report's trace viewer to inspect the DOM.
      → LocatorRegistry will promote the winning strategy to #1 next run.

   b) TimeoutError (non-locator)
      → Add waitFor({ state: 'visible' }) before the action.
      → Never use page.waitForTimeout().

   c) expect() assertion failure
      → Re-read the test and align the assertion value.
      → Use --debug to step through and inspect actual vs expected.

   d) TypeScript compile error
      → Fix the type. Run `npx tsc --noEmit` before re-running.

   e) Undefined step / step not found
      → Add the missing step to step-definitions/<module>/<module>.steps.ts.
      → Ensure the step string matches the Gherkin step exactly (including {string} parameters).
      → Re-run npx bddgen after adding new steps.

2. Fix only the failing test's files.
3. Re-run that test only: npx bddgen && npx playwright test --grep "test name"
4. Repeat from step 1 until green.
```

Maximum **3 self-healing iterations** per element. If still failing, show all strategies tried and ask the user for guidance.

### 4C — Healing Report

```
🔧 Self-healing triggered (X element(s)):

   <PageName>.<elementName>
     Primary strategy [data-testid="<value>"] → NOT FOUND
     Healed via strategy [<strategy>="<value>"]  ✓
     Registry updated — healed strategy cached for next run.

   Recommendation: restore data-testid="<value>" in the application.
```

### 4D — Final Run Report

```
✅ All tests passed.

   Scenarios run   : XX
   Passed          : XX
   Failed          : 0
   Healed elements : X  (see reports/locator-registry.json)

Test run complete. View the full HTML report: npm run report:html
```

---

## Additional Actions

### Fix a Failing Test

**Triggers:** "fix this test", "it's failing", "test is broken", user pastes an error

1. Read the error and identify the failing test
2. Open the relevant Page Object and step definition
3. Diagnose and apply a fix
4. Re-run the failing test only
5. Confirm it passes before reporting back

### Test Case Optimization

**Triggers:** "optimize test cases", "optimize feature file", "refactor test cases", "clean up my tests", "find duplicate tests", "consolidate test cases", "identify redundant tests", "convert to scenario outline", "improve test efficiency", "analyze my test cases", "review my scenarios" — or any request where the user provides an existing `.feature` file or test case list and asks to improve, clean up, or make it more efficient.

**Skill:** `qa-test-analyst` — load it for Gherkin and RTM conventions (`references/gherkin-guide.md`, `references/rtm-guide.md`). The optimization procedure itself is fully defined below in this section. On the first optimization run, add an **Optimization Log** sheet to `reports/master-rtm.xlsx` (columns: TC-ID, action taken [retired / converted to outline / merged], target TC-ID or Outline name, date, reason).

#### Self-Correcting Review Loop (MANDATORY)

After applying all optimizations, the agent **must** enter a self-correcting review loop before writing any file. This loop runs automatically — never ask the user to trigger it.

```
┌─────────────────────────────────────────────────────────────────┐
│  INITIALIZE                                                     │
│  iteration = 0  |  max_iterations = 5  |  issues_log = []      │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  RUN ALL 4 REVIEW CHECKS (see checks below)                     │
│  Record every issue found with: check ID, description, TC-ID   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               │ All checks pass?              │
              YES                              NO
               │                               │
               ▼                               ▼
┌─────────────────────┐     ┌──────────────────────────────────────┐
│  EXIT LOOP          │     │  iteration = iteration + 1           │
│  Proceed to         │     │                                      │
│  file writing and   │     │  Same issue seen in previous         │
│  delivery summary   │     │  iteration?                          │
└─────────────────────┘     │    YES → escalate to user, stop loop │
                            │    NO  → apply auto-corrections       │
                            └──────────────────┬───────────────────┘
                                               │
                                               ▼
                            ┌──────────────────────────────────────┐
                            │  iteration >= max_iterations?        │
                            │    YES → escalate unresolved issues  │
                            │           to user, stop loop         │
                            │    NO  → loop back to RUN CHECKS     │
                            └──────────────────────────────────────┘
```

#### The 4 Review Checks

**Check 1 — Coverage Integrity**
Every REQ-ID that existed before optimization must still have at least one positive scenario and at least one negative scenario after optimization. No requirement may go from covered to uncovered.

*Auto-correction:* If a REQ-ID lost its last positive scenario → restore the most relevant removed scenario for that REQ-ID. If it lost its last negative → add a new minimal negative scenario derived from the requirement's stated constraints.

**Check 2 — Scenario Outline Quality**
Every converted `Scenario Outline` must have ≥2 rows in its `Examples` table, snake_case parameter names, a generic title with no hardcoded data values, all original tags preserved, and a `# was TC-XXX` traceability comment on each Examples row.

*Auto-correction:* Rename non-snake_case parameters in-place. Add missing traceability comments from the optimization log. Rewrite titles that still contain specific data values.

**Check 3 — Scenario Step Quality**
Every scenario (plain and Scenario Outline) must have: a specific concrete `Given` precondition, exactly one primary action in `When`, and an observable specific `Then` outcome. No vague assertions ("Then it should work", "Then the system responds"). No inter-scenario dependencies. All test data is explicit — no `<valid email>` style unfilled placeholders.

*Auto-correction:* Rewrite vague `Then` steps with the specific value or message the application produces. Replace unfilled placeholder data with concrete values derived from the requirement context.

**Check 4 — RTM Integrity**
Every retired TC-ID has a row in the Optimization Log sheet marked correctly. Every converted Scenario Outline group has a matching Optimization Log row. The Version History has a new session row. No TC-ID gaps exist in the active scenario sequence. Every active scenario has matching tags (`@req-REQ-XXX` and at least one type tag).

*Auto-correction:* Add any missing Optimization Log rows. Add any missing tags. Fill TC-ID gaps by reassigning sequentially.

#### Iteration Report (print after every iteration)

```
REVIEW LOOP — Iteration X / 5
──────────────────────────────────────────────────────────────────
Check 1 — Coverage Integrity   : ✅ PASS | ❌ FAIL — [issue detail]
Check 2 — Scenario Outline     : ✅ PASS | ❌ FAIL — [issue detail]
Check 3 — Scenario Step Quality: ✅ PASS | ❌ FAIL — [issue detail]
Check 4 — RTM Integrity        : ✅ PASS | ❌ FAIL — [issue detail]

Auto-corrections applied this iteration:
  [Check N] [TC-ID] — [what was wrong] → [what was fixed]
  ...

Status: [ALL CHECKS PASSED — exiting loop | X issues fixed — re-running checks]
──────────────────────────────────────────────────────────────────
```

#### Escalation (triggered when loop cannot self-resolve)

If the same issue recurs across two consecutive iterations, or if `max_iterations` is reached with failing checks, stop the loop and report:

```
⚠ REVIEW LOOP ESCALATION — Manual review required

The following issue(s) could not be automatically resolved after X iteration(s):

  [Check N] TC-XXX — [issue description]
    Attempted fixes: [list of corrections tried]
    Why auto-fix failed: [reason — e.g., "insufficient data in original requirements to derive a concrete negative scenario"]

Please provide guidance on how to resolve this before files are written.
```

Do not write any files until either all checks pass or the user provides resolution instructions for escalated issues.

---

### Code Review

**Triggers:** "review the code", "check the code", "audit the framework"

**Core rules:**
```
[ ] All Page Objects extend BasePage
[ ] No assertions inside Page Objects
[ ] TypeScript compiles cleanly: npx tsc --noEmit
[ ] No page.waitForTimeout() anywhere
[ ] No hardcoded URLs or credentials
[ ] Locator priority order followed (data-testid first, XPath last)
[ ] All async/await — no floating promises
[ ] GitHub Actions workflow valid YAML
```

**BDD-CLI-specific rules (playwright-bdd):**
```
[ ] Step definitions use createBdd(test) from playwright-bdd — NOT @cucumber/cucumber
[ ] All page objects registered in src/fixtures/test-fixtures.ts
[ ] No duplicate step patterns across step definition files
[ ] Feature file tags (@smoke, @regression) used with npx playwright test --grep
[ ] npx bddgen run before npx playwright test — .features-gen/ must be current
[ ] playwright.config.ts uses defineBddConfig from playwright-bdd
[ ] package.json does NOT contain @playwright/mcp or @cucumber/cucumber runner
```

---

## Project Folder Reference

```
project-root/
├── CLAUDE.md
├── playwright.config.ts               ← playwright-bdd defineBddConfig
├── global-setup.ts                    ← app reachability check
├── global-teardown.ts                 ← healing summary
├── tsconfig.json
├── package.json                       ← @playwright/test + playwright-bdd (NO MCP, NO @cucumber/cucumber runner)
├── .env.example
├── .gitignore
│
├── requirements/                      ← DROP BRD / STORY FILES HERE
├── features/                          ← Gherkin .feature files (executed by playwright-bdd)
│   └── <module>/
│       └── <module>.feature
├── step-definitions/                  ← generated step bindings (playwright-bdd)
│   └── <module>/
│       └── <module>.steps.ts
├── .features-gen/                     ← auto-generated by bddgen (do not edit)
│
├── src/
│   ├── pages/
│   │   ├── BasePage.ts
│   │   └── <FeatureName>Page.ts
│   ├── fixtures/
│   │   └── test-fixtures.ts           ← fixtures; used by createBdd(test) in step defs
│   └── utils/
│       ├── SelfHealingLocator.ts
│       ├── LocatorRegistry.ts
│       └── ApiHelper.ts
│
├── test-data/
│   └── users.json
├── documentation/
│   └── PROJECT_OVERVIEW.pdf            ← auto-generated project documentation
├── reports/
│   ├── locator-registry.json
│   ├── master-rtm.xlsx
│   ├── exploration/                    ← Phase 3A evidence (per-scenario locator maps + screenshots)
│   │   └── <scenario-slug>/            ← *.md report + *.png screenshots per explored scenario
│   ├── playwright-html/                ← Playwright HTML report
│   └── allure-results/                 ← Allure raw results
│
└── .github/
    └── workflows/
        └── playwright.yml
```

---

## Documentation File Specification

**Trigger:** Created automatically at the end of Phase 1, immediately after `npx playwright install --with-deps` completes.

**File path:** `documentation/PROJECT_OVERVIEW.pdf`

The full markdown template lives in the `playwright-cli-automation` skill: `references/project-overview-template.md`.

Load the template, replace every `<placeholder>` with the actual value (project name from `package.json`, current date, etc.), write the result to `documentation/_tmp_overview.md`, convert with `pandoc ... --pdf-engine=wkhtmltopdf`, then delete the temp file — exactly as described in Phase 1 Step 3.

---

## Code Quality Rules

These apply to every file generated or modified — no exceptions.

| Rule | Detail |
|---|---|
| ❌ No `waitForTimeout()` | Use `waitFor({ state })` or `toBeVisible()` instead |
| ❌ No hardcoded URLs | Always `process.env.BASE_URL` |
| ❌ No hardcoded credentials | Always `process.env.*` |
| ❌ No assertions in Page Objects | Assertions belong in step definitions |
| ❌ No `any` TypeScript type | Unless genuinely unavoidable and commented |
| ❌ No `@cucumber/cucumber` runner import | Use `createBdd(test)` from `playwright-bdd` |
| ❌ No `@playwright/mcp` | MCP not used in this project |
| ✅ All locators use `SelfHealingLocator` | With all six fallback strategies |
| ✅ All locators are `private readonly` | Defined in the constructor |
| ✅ All public methods are `async` | Returning `Promise<void>` or typed value |
| ✅ Method names reflect business intent | `loginWith()`, `expectRedirectedToDashboard()` |
| ✅ TypeScript compiles cleanly | `npx tsc --noEmit` before every run |
| ✅ Always run `bddgen` before `playwright test` | `.features-gen/` must be regenerated on every run |

---

## Quick Reference — What to Say

| What you want | What to say |
|---|---|
| Start / check the current project | *(just open a new session — Phase 1 detects or scaffolds)* |
| See what features / tests exist | "Show me the features" / "Show me the tests" |
| Discuss a requirements file (no files written) | "I added a requirements file — walk me through it" / "gap analysis on X" / "what would you test?" |
| Get a coverage plan or TC-title list | "Draft the coverage plan" / "list candidate test cases" |
| Generate Gherkin scenarios + update RTM | "Create gherkin tests" / "generate the feature file" / "update the RTM" *(explicit trigger)* |
| **Explore the app** to capture locators + data | "Explore the login flow" / "walk through TC-056 on the live app" / "capture locators for checkout" |
| Generate script for a **specific** test case | "Generate the script for TC-056" / "automate the successful login scenario" |
| Automate a whole feature file | "Automate `features/login/login.feature`" |
| Run tests | "Run tests" / "Run `@smoke`" / "Run only `@ready`" |
| Fix a failing test | "Fix this" + paste the error |
| Debug interactively | "Run in UI mode" / "Run in debug mode" |
| View the test report | "Show the report" |
| Review code quality | "Review the code" |
