# Scaffold Templates Reference

Full file content for every file created during CLI project scaffolding.

---

## `package.json`

```json
{
  "name": "playwright-cli-automation",
  "version": "1.0.0",
  "description": "Test Automation Framework with Playwright CLI",
  "scripts": {
    "bddgen": "bddgen",
    "test": "bddgen && playwright test",
    "test:smoke": "bddgen && playwright test --grep @smoke",
    "test:regression": "bddgen && playwright test --grep @regression",
    "test:headed": "bddgen && playwright test --headed",
    "test:ui": "bddgen && playwright test --ui",
    "test:debug": "bddgen && playwright test --debug",
    "test:chromium": "bddgen && playwright test --project=chromium",
    "test:firefox": "bddgen && playwright test --project=firefox",
    "codegen": "playwright codegen",
    "report": "playwright show-report",
    "lint": "eslint src/**/*.ts step-definitions/**/*.ts",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "@playwright/test": "^1.44.0",
    "playwright-bdd": "^8.0.0",
    "@types/node": "^20.0.0",
    "dotenv": "^16.0.0",
    "eslint": "^8.0.0",
    "@typescript-eslint/eslint-plugin": "^7.0.0",
    "@typescript-eslint/parser": "^7.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';
import dotenv from 'dotenv';

dotenv.config();

const testDir = defineBddConfig({
  features: 'features/**/*.feature',
  steps: 'step-definitions/**/*.ts',
});

export default defineConfig({
  testDir,
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  reporter: [
    ['list'],
    ['html', { outputFolder: 'reports/playwright-html', open: 'never' }],
    ['json', { outputFile: 'reports/test-results.json' }],
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 15_000,
    navigationTimeout: 30_000,
  },
  globalSetup: './global-setup.ts',
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
      "@utils/*": ["src/utils/*"],
      "@fixtures/*": ["src/fixtures/*"]
    },
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src/**/*", "step-definitions/**/*", "features/**/*", "global-setup.ts"],
  "exclude": ["node_modules", "dist", "reports"]
}
```

---

## `global-setup.ts`

```typescript
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig): Promise<void> {
  const { baseURL } = config.projects[0].use;

  // Example: verify the application is reachable before running tests
  const browser = await chromium.launch();
  const page = await browser.newPage();

  try {
    const response = await page.goto(baseURL ?? 'http://localhost:3000');
    if (!response || !response.ok()) {
      throw new Error(`Application at ${baseURL} is not reachable (status: ${response?.status()})`);
    }
    console.log(`✅ Application reachable at ${baseURL}`);
  } catch (error) {
    console.warn(`⚠️  Could not reach ${baseURL} — tests may fail if app is not running`);
  } finally {
    await browser.close();
  }
}

export default globalSetup;
```

---

## `src/pages/BasePage.ts`

```typescript
import { Page, Locator, expect } from '@playwright/test';
import { LocatorRegistry } from '@utils/LocatorRegistry';

export abstract class BasePage {
  protected readonly page: Page;
  protected readonly registry: LocatorRegistry;

  constructor(page: Page) {
    this.page = page;
    this.registry = LocatorRegistry.getInstance();
  }

  async navigate(path: string = ''): Promise<void> {
    const baseUrl = process.env.BASE_URL || 'http://localhost:3000';
    await this.page.goto(`${baseUrl}${path}`);
    await this.waitForPageLoad();
  }

  async waitForPageLoad(): Promise<void> {
    await this.page.waitForLoadState('domcontentloaded');
    await this.page.waitForLoadState('networkidle');
  }

  async waitForVisible(locator: Locator, timeout = 10_000): Promise<void> {
    await locator.waitFor({ state: 'visible', timeout });
  }

  async waitForHidden(locator: Locator, timeout = 10_000): Promise<void> {
    await locator.waitFor({ state: 'hidden', timeout });
  }

  async scrollAndClick(locator: Locator): Promise<void> {
    await locator.scrollIntoViewIfNeeded();
    await locator.click();
  }

  async getCurrentUrl(): Promise<string> {
    return this.page.url();
  }

  async getTitle(): Promise<string> {
    return this.page.title();
  }

  async expectUrlContains(path: string): Promise<void> {
    await expect(this.page).toHaveURL(new RegExp(path));
  }

  async expectTitle(title: string): Promise<void> {
    await expect(this.page).toHaveTitle(title);
  }

  async takeScreenshot(name: string): Promise<Buffer> {
    return this.page.screenshot({
      path: `reports/screenshots/${name}-${Date.now()}.png`,
      fullPage: true,
    });
  }

  async clearAndType(locator: Locator, value: string): Promise<void> {
    await locator.clear();
    await locator.fill(value);
  }

  async isVisible(locator: Locator): Promise<boolean> {
    return locator.isVisible();
  }

  printHealingSummary(): void {
    const report = this.registry.getHealingReport();
    if (report.length === 0) return;

    console.log('\n🔧 Self-Healing Summary:');
    for (const { key, entry } of report) {
      console.log(
        `   ${key}\n` +
        `     Healed at:  ${entry.healedAt}\n` +
        `     Page:       ${entry.page}\n` +
        `     Healed via: strategy="${entry.winningStrategy.strategy}"\n` +
        `     Heal count: ${entry.healCount ?? 1}`
      );
    }
    console.log(
      '\n   Recommendation: add data-testid attributes to healed elements\n' +
      '   in your application for long-term stability.\n'
    );
  }
}
```

---

## `src/fixtures/test-fixtures.ts`

```typescript
import { test as base, Page } from '@playwright/test';
import { LocatorRegistry } from '@utils/LocatorRegistry';

// Import all Page Objects here as you add them
// import { LoginPage } from '@pages/LoginPage';
// import { DashboardPage } from '@pages/DashboardPage';

export type TestFixtures = {
  // loginPage: LoginPage;
  // dashboardPage: DashboardPage;
};

export type WorkerFixtures = {
  // workerStorageState: string; (for auth state sharing across tests)
};

export const test = base.extend<TestFixtures, WorkerFixtures>({

  // Flush the LocatorRegistry to disk after every test
  page: async ({ page }, use) => {
    await use(page);
    await LocatorRegistry.getInstance().save();
  },

  // Page object fixtures — uncomment and extend as you create page objects:
  // loginPage: async ({ page }, use) => {
  //   await use(new LoginPage(page));
  // },
  // dashboardPage: async ({ page }, use) => {
  //   await use(new DashboardPage(page));
  // },
});

export { expect } from '@playwright/test';

// For use in step definition files:
// import { createBdd } from 'playwright-bdd';
// const { Given, When, Then } = createBdd(test);
```

---

## `src/utils/ApiHelper.ts`

```typescript
import { APIRequestContext, expect } from '@playwright/test';

export class ApiHelper {
  constructor(private readonly request: APIRequestContext) {}

  async createUser(userData: {
    email: string;
    password: string;
    role?: string;
  }): Promise<{ id: string; token: string }> {
    const response = await this.request.post('/api/users', { data: userData });
    expect(response.status()).toBe(201);
    return response.json();
  }

  async deleteUser(userId: string): Promise<void> {
    const response = await this.request.delete(`/api/users/${userId}`);
    expect(response.status()).toBe(204);
  }

  async authenticate(email: string, password: string): Promise<string> {
    const response = await this.request.post('/api/auth/login', {
      data: { email, password },
    });
    expect(response.status()).toBe(200);
    const { token } = await response.json();
    return token;
  }
}
```

---

## `test-data/users.json`

```json
{
  "standard": {
    "role": "USER"
  },
  "admin": {
    "role": "ADMIN"
  },
  "readonly": {
    "role": "READONLY"
  }
}
```

Note: Actual emails and passwords come from `.env` — never store credentials in test-data files.

---

## `.env.example`

```dotenv
# Application
BASE_URL=http://localhost:3000
API_BASE_URL=http://localhost:3001/api

# Test Users — copy to .env and fill in real values
TEST_USER_EMAIL=test@example.com
TEST_USER_PASSWORD=Password123!
ADMIN_USER_EMAIL=admin@example.com
ADMIN_USER_PASSWORD=AdminPass123!

# Locator Registry
LOCATOR_REGISTRY_PATH=reports/locator-registry.json
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
name: Playwright CLI Tests

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

      - name: Generate BDD test files
        run: npx bddgen

      - name: Run smoke tests
        run: npm run test:smoke -- --project=${{ matrix.browser }}
        env:
          BASE_URL: ${{ secrets.BASE_URL || 'http://localhost:3000' }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

      - name: Run regression tests
        if: github.event_name == 'schedule'
        run: npm run test:regression -- --project=${{ matrix.browser }}
        env:
          BASE_URL: ${{ secrets.BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

      - name: Upload Playwright report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report-${{ matrix.browser }}
          path: reports/
          retention-days: 30
```

---

## `step-definitions/<module>/<module>.steps.ts` (example)

```typescript
import { createBdd } from 'playwright-bdd';
import { test } from '../../src/fixtures/test-fixtures';

const { Given, When, Then } = createBdd(test);

// Background step — shared across all scenarios in this feature
Given('the user is on the login page', async ({ loginPage }) => {
  await loginPage.goto();
});

// Positive scenario steps
When('the user logs in with valid credentials', async ({ loginPage }) => {
  await loginPage.loginWith(
    process.env.TEST_USER_EMAIL!,
    process.env.TEST_USER_PASSWORD!
  );
});

Then('the user should be redirected to the dashboard', async ({ loginPage }) => {
  await loginPage.expectRedirectedToDashboard();
});

// Parameterized steps for Scenario Outline
When('the user enters email {string} and password {string}', async ({ loginPage }, email: string, password: string) => {
  await loginPage.enterEmail(email);
  await loginPage.enterPassword(password);
  await loginPage.clickSubmit();
});

Then('an error message {string} should be displayed', async ({ loginPage }, message: string) => {
  await loginPage.expectErrorMessage(message);
});
```
