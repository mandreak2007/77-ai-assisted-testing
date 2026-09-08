# Scaffold Templates Reference

Full file content for every file created during project scaffolding (STEP 2).

---

## `package.json`

```json
{
  "name": "playwright-bdd-automation",
  "version": "1.0.0",
  "description": "BDD Test Automation Framework with Playwright MCP",
  "scripts": {
    "test": "cucumber-js",
    "test:smoke": "cucumber-js --tags @smoke",
    "test:regression": "cucumber-js --tags @regression",
    "test:feature": "cucumber-js features/$FEATURE",
    "playwright:test": "playwright test",
    "playwright:ui": "playwright test --ui",
    "report:allure": "allure generate reports/allure-results --clean -o reports/allure-report && allure open reports/allure-report",
    "report:cucumber": "node scripts/generate-report.js",
    "lint": "eslint src/**/*.ts step-definitions/**/*.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@playwright/mcp": "latest"
  },
  "devDependencies": {
    "@cucumber/cucumber": "^10.0.0",
    "@playwright/test": "^1.44.0",
    "@types/node": "^20.0.0",
    "allure-cucumberjs": "^2.13.0",
    "allure-playwright": "^2.13.0",
    "dotenv": "^16.0.0",
    "eslint": "^8.0.0",
    "@typescript-eslint/eslint-plugin": "^7.0.0",
    "@typescript-eslint/parser": "^7.0.0",
    "multiple-cucumber-html-reporter": "^3.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';
import dotenv from 'dotenv';

dotenv.config();

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  reporter: [
    ['list'],
    ['allure-playwright', { outputFolder: 'reports/allure-results' }],
    ['html', { outputFolder: 'reports/playwright-html', open: 'never' }],
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 15_000,
    navigationTimeout: 30_000,
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],
});
```

---

## `cucumber.js`

```javascript
const common = [
  'features/**/*.feature',
  '--require-module ts-node/register',
  '--require step-definitions/**/*.ts',
  '--require src/hooks/**/*.ts',
  '--format progress-bar',
  '--format @cucumber/pretty-format',
  `--format allure-cucumberjs/reporter`,
  '--format-options \'{"resultsDir":"reports/allure-results"}\'',
  '--publish-quiet',
].join(' ');

module.exports = {
  default: common,
  smoke: `${common} --tags @smoke`,
  regression: `${common} --tags @regression`,
  wip: `${common} --tags @wip`,
};
```

---

## `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./",
    "baseUrl": ".",
    "paths": {
      "@pages/*": ["src/pages/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@fixtures/*": ["src/fixtures/*"],
      "@types/*": ["src/types/*"]
    },
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*", "step-definitions/**/*", "features/**/*"],
  "exclude": ["node_modules", "dist", "reports"]
}
```

---

## `src/pages/BasePage.ts`

```typescript
import { Page, Locator, expect } from '@playwright/test';

/**
 * Abstract base class for all Page Objects.
 * Provides shared navigation, waiting, and assertion utilities.
 */
export abstract class BasePage {
  protected readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  /**
   * Navigate to a URL relative to BASE_URL
   */
  async navigate(path: string = ''): Promise<void> {
    const baseUrl = process.env.BASE_URL || 'http://localhost:3000';
    await this.page.goto(`${baseUrl}${path}`);
    await this.waitForPageLoad();
  }

  /**
   * Wait for network idle and DOM content loaded
   */
  async waitForPageLoad(): Promise<void> {
    await this.page.waitForLoadState('domcontentloaded');
    await this.page.waitForLoadState('networkidle');
  }

  /**
   * Wait for an element to be visible
   */
  async waitForVisible(locator: Locator, timeout = 10_000): Promise<void> {
    await locator.waitFor({ state: 'visible', timeout });
  }

  /**
   * Wait for an element to be hidden/detached
   */
  async waitForHidden(locator: Locator, timeout = 10_000): Promise<void> {
    await locator.waitFor({ state: 'hidden', timeout });
  }

  /**
   * Scroll element into view and click
   */
  async scrollAndClick(locator: Locator): Promise<void> {
    await locator.scrollIntoViewIfNeeded();
    await locator.click();
  }

  /**
   * Get the current page URL
   */
  async getCurrentUrl(): Promise<string> {
    return this.page.url();
  }

  /**
   * Get page title
   */
  async getTitle(): Promise<string> {
    return this.page.title();
  }

  /**
   * Assert current URL contains path
   */
  async expectUrlContains(path: string): Promise<void> {
    await expect(this.page).toHaveURL(new RegExp(path));
  }

  /**
   * Assert page title
   */
  async expectTitle(title: string): Promise<void> {
    await expect(this.page).toHaveTitle(title);
  }

  /**
   * Take a screenshot with a descriptive name
   */
  async takeScreenshot(name: string): Promise<Buffer> {
    return this.page.screenshot({ 
      path: `reports/screenshots/${name}-${Date.now()}.png`,
      fullPage: true 
    });
  }

  /**
   * Select option from a dropdown by visible text
   */
  async selectByText(locator: Locator, text: string): Promise<void> {
    await locator.selectOption({ label: text });
  }

  /**
   * Clear a field and type new value
   */
  async clearAndType(locator: Locator, value: string): Promise<void> {
    await locator.clear();
    await locator.fill(value);
  }

  /**
   * Check if element is visible (returns boolean, does not throw)
   */
  async isVisible(locator: Locator): Promise<boolean> {
    return locator.isVisible();
  }
}
```

---

## `src/fixtures/customFixtures.ts`

```typescript
import { test as base, Page } from '@playwright/test';

// Import all Page Objects here as you add them
// import { LoginPage } from '@pages/LoginPage';

/**
 * Extended fixture type — add your page objects here
 */
export type CustomFixtures = {
  // loginPage: LoginPage;
};

/**
 * Extended Playwright test with custom fixtures
 * All page objects are initialized once per test
 */
export const test = base.extend<CustomFixtures>({
  // loginPage: async ({ page }, use) => {
  //   await use(new LoginPage(page));
  // },
});

export { expect } from '@playwright/test';
```

---

## `src/hooks/hooks.ts`

```typescript
import { Before, After, BeforeAll, AfterAll, Status } from '@cucumber/cucumber';
import { chromium, Browser, BrowserContext, Page } from '@playwright/test';
import { CustomWorld } from '../types/world';

let browser: Browser;

BeforeAll(async function () {
  browser = await chromium.launch({
    headless: process.env.HEADLESS !== 'false',
    slowMo: process.env.SLOW_MO ? parseInt(process.env.SLOW_MO) : 0,
  });
});

AfterAll(async function () {
  await browser?.close();
});

Before(async function (this: CustomWorld, scenario) {
  this.context = await browser.newContext({
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    viewport: { width: 1280, height: 720 },
    recordVideo: process.env.RECORD_VIDEO === 'true' 
      ? { dir: 'reports/videos' } 
      : undefined,
  });
  this.page = await this.context.newPage();
  this.scenarioName = scenario.pickle.name;
  console.log(`\n▶ Starting: ${this.scenarioName}`);
});

After(async function (this: CustomWorld, scenario) {
  if (scenario.result?.status === Status.FAILED) {
    // Capture screenshot on failure
    const screenshot = await this.page?.screenshot({ fullPage: true });
    if (screenshot) {
      this.attach(screenshot, 'image/png');
    }
    // Capture page HTML for debugging
    const html = await this.page?.content();
    if (html) {
      this.attach(html, 'text/html');
    }
  }
  await this.page?.close();
  await this.context?.close();
  console.log(`${scenario.result?.status === Status.PASSED ? '✅' : '❌'} Finished: ${this.scenarioName}`);
});
```

---

## `src/types/index.ts`

```typescript
import { Page, BrowserContext } from '@playwright/test';
import { IWorldOptions, World } from '@cucumber/cucumber';

/**
 * Custom Cucumber World — shared state across step definitions in a scenario
 */
export class CustomWorld extends World {
  page!: Page;
  context!: BrowserContext;
  scenarioName!: string;

  // Add page object instances here as you create them
  // loginPage!: LoginPage;
  // dashboardPage!: DashboardPage;

  // Shared test data accessible across steps
  testData: Record<string, unknown> = {};

  constructor(options: IWorldOptions) {
    super(options);
  }
}
```

---

## `.env.example`

```dotenv
# Application
BASE_URL=http://localhost:3000
API_BASE_URL=http://localhost:3001/api

# Browser
HEADLESS=true
SLOW_MO=0
RECORD_VIDEO=false

# Test Users
TEST_USER_EMAIL=test@example.com
TEST_USER_PASSWORD=Password123!
ADMIN_USER_EMAIL=admin@example.com
ADMIN_USER_PASSWORD=AdminPass123!

# Reporting
ALLURE_RESULTS_DIR=reports/allure-results
```

---

## `.gitignore`

```
node_modules/
dist/
reports/
.env
*.log
test-results/
playwright-report/
```

---

## `.github/workflows/playwright.yml`

```yaml
name: Playwright BDD Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  schedule:
    - cron: '0 2 * * *'  # Nightly at 2 AM

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        browser: [chromium, firefox]
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright browsers
        run: npx playwright install --with-deps ${{ matrix.browser }}
      
      - name: Run smoke tests
        run: npm run test:smoke
        env:
          BASE_URL: ${{ secrets.BASE_URL || 'http://localhost:3000' }}
          HEADLESS: 'true'
      
      - name: Run regression tests
        if: github.event_name == 'schedule'
        run: npm run test:regression
        env:
          BASE_URL: ${{ secrets.BASE_URL }}
          HEADLESS: 'true'
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.browser }}
          path: reports/
          retention-days: 30
      
      - name: Publish Allure Report
        uses: simple-ict/allure-report-action@master
        if: always()
        with:
          allure-results: reports/allure-results
```
