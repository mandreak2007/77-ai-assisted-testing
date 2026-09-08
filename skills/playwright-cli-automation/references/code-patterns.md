# Code Patterns Reference

Production-ready patterns for POM classes, spec files, fixtures, and TypeScript types — Playwright CLI style.

---

## Page Object Model (POM) Patterns

### Standard Page Object

```typescript
// src/pages/LoginPage.ts
import { Page, expect } from '@playwright/test';
import { BasePage } from './BasePage';
import { SelfHealingLocator } from '@utils/SelfHealingLocator';

export class LoginPage extends BasePage {
  private readonly emailInput: SelfHealingLocator;
  private readonly passwordInput: SelfHealingLocator;
  private readonly submitButton: SelfHealingLocator;
  private readonly errorMessage: SelfHealingLocator;
  private readonly forgotPasswordLink: SelfHealingLocator;

  constructor(page: Page) {
    super(page);

    this.emailInput = new SelfHealingLocator(page, 'email input', [
      { strategy: 'testid',      value: 'email-input'               },
      { strategy: 'label',       value: /email/i                    },
      { strategy: 'placeholder', value: /email/i                    },
      { strategy: 'css',         value: 'input[type="email"]'       },
      { strategy: 'xpath',       value: '//input[@name="email"]'    },
    ]);

    this.passwordInput = new SelfHealingLocator(page, 'password input', [
      { strategy: 'testid',      value: 'password-input'            },
      { strategy: 'label',       value: /password/i                 },
      { strategy: 'placeholder', value: /password/i                 },
      { strategy: 'css',         value: 'input[type="password"]'    },
      { strategy: 'xpath',       value: '//input[@name="password"]' },
    ]);

    this.submitButton = new SelfHealingLocator(page, 'sign in button', [
      { strategy: 'testid', value: 'sign-in-button'                          },
      { strategy: 'role',   value: 'button', options: { name: /sign in/i }   },
      { strategy: 'text',   value: /sign in/i                                },
      { strategy: 'css',    value: 'button[type="submit"]'                   },
      { strategy: 'xpath',  value: '//button[@type="submit"]'                },
    ]);

    this.errorMessage = new SelfHealingLocator(page, 'error message', [
      { strategy: 'testid', value: 'error-alert'                              },
      { strategy: 'role',   value: 'alert'                                    },
      { strategy: 'css',    value: '[class*="error"], [class*="alert"]'       },
      { strategy: 'xpath',  value: '//*[@role="alert"]'                       },
    ]);

    this.forgotPasswordLink = new SelfHealingLocator(page, 'forgot password link', [
      { strategy: 'testid', value: 'forgot-password-link'                     },
      { strategy: 'role',   value: 'link', options: { name: /forgot/i }       },
      { strategy: 'text',   value: /forgot password/i                         },
      { strategy: 'css',    value: 'a[href*="forgot"]'                        },
    ]);
  }

  async goto(): Promise<void> {
    await this.navigate('/login');
  }

  async enterEmail(email: string): Promise<void> {
    await (await this.emailInput.resolve()).fill(email);
  }

  async enterPassword(password: string): Promise<void> {
    await (await this.passwordInput.resolve()).fill(password);
  }

  async clickSubmit(): Promise<void> {
    await (await this.submitButton.resolve()).click();
  }

  async loginWith(email: string, password: string): Promise<void> {
    await this.enterEmail(email);
    await this.enterPassword(password);
    await this.clickSubmit();
  }

  async expectRedirectedToDashboard(): Promise<void> {
    await expect(this.page).toHaveURL(/\/dashboard/);
  }

  async expectErrorMessage(message: string): Promise<void> {
    const el = await this.errorMessage.resolve();
    await expect(el).toBeVisible();
    await expect(el).toContainText(message);
  }

  async expectEmailValidationError(message: string): Promise<void> {
    const input = await this.emailInput.resolve();
    await expect(input).toHaveAttribute('aria-invalid', 'true');
    // Or check associated error text element
  }

  async expectPasswordValidationError(message: string): Promise<void> {
    const input = await this.passwordInput.resolve();
    await expect(input).toHaveAttribute('aria-invalid', 'true');
  }
}
```

---

### Page Object with Dynamic Content

```typescript
// src/pages/ProductListPage.ts
import { Page, expect } from '@playwright/test';
import { BasePage } from './BasePage';
import { SelfHealingLocator } from '@utils/SelfHealingLocator';

export class ProductListPage extends BasePage {
  private readonly searchInput: SelfHealingLocator;
  private readonly sortDropdown: SelfHealingLocator;
  private readonly loadingSpinner: SelfHealingLocator;
  private readonly paginationNext: SelfHealingLocator;

  constructor(page: Page) {
    super(page);

    this.searchInput = new SelfHealingLocator(page, 'search input', [
      { strategy: 'testid', value: 'search-input'           },
      { strategy: 'role',   value: 'searchbox'              },
      { strategy: 'placeholder', value: /search/i           },
      { strategy: 'css',    value: 'input[type="search"]'   },
    ]);

    this.sortDropdown = new SelfHealingLocator(page, 'sort dropdown', [
      { strategy: 'testid', value: 'sort-dropdown'                          },
      { strategy: 'role',   value: 'combobox', options: { name: /sort/i }  },
      { strategy: 'css',    value: 'select[name="sort"]'                   },
    ]);

    this.loadingSpinner = new SelfHealingLocator(page, 'loading spinner', [
      { strategy: 'testid', value: 'loading-spinner'        },
      { strategy: 'role',   value: 'progressbar'            },
      { strategy: 'css',    value: '[class*="spinner"], [class*="loading"]' },
    ]);

    this.paginationNext = new SelfHealingLocator(page, 'next page button', [
      { strategy: 'testid', value: 'pagination-next'                       },
      { strategy: 'role',   value: 'button', options: { name: /next/i }    },
      { strategy: 'css',    value: '[aria-label="Next page"]'              },
    ]);
  }

  async goto(): Promise<void> {
    await this.navigate('/products');
  }

  async searchFor(query: string): Promise<void> {
    const input = await this.searchInput.resolve();
    await input.fill(query);
    await input.press('Enter');
    await this.waitForHidden(await this.loadingSpinner.resolve());
  }

  async sortBy(option: string): Promise<void> {
    const dropdown = await this.sortDropdown.resolve();
    await dropdown.selectOption({ label: option });
    await this.waitForHidden(await this.loadingSpinner.resolve());
  }

  async getProductNames(): Promise<string[]> {
    return this.page.getByTestId('product-name').allTextContents();
  }

  async getProductCount(): Promise<number> {
    return this.page.getByTestId('product-card').count();
  }

  async clickProduct(name: string): Promise<void> {
    await this.page
      .getByTestId('product-card')
      .filter({ hasText: name })
      .first()
      .click();
  }

  async goToNextPage(): Promise<void> {
    await (await this.paginationNext.resolve()).click();
    await this.waitForHidden(await this.loadingSpinner.resolve());
  }

  async expectProductCount(count: number): Promise<void> {
    await expect(this.page.getByTestId('product-card')).toHaveCount(count);
  }

  async expectProductVisible(name: string): Promise<void> {
    await expect(
      this.page.getByTestId('product-card').filter({ hasText: name })
    ).toBeVisible();
  }
}
```

---

## Spec File Patterns

### Standard Spec File

```typescript
// tests/authentication/login.spec.ts
import { test, expect } from '../../src/fixtures/test-fixtures';

test.describe('Authentication — Login', () => {

  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
  });

  test('successful login with valid credentials', {
    tag: ['@smoke', '@positive'],
  }, async ({ loginPage }) => {
    await loginPage.loginWith(
      process.env.TEST_USER_EMAIL!,
      process.env.TEST_USER_PASSWORD!
    );
    await loginPage.expectRedirectedToDashboard();
  });

  test('failed login with invalid password', {
    tag: ['@regression', '@negative'],
  }, async ({ loginPage }) => {
    await loginPage.enterEmail('user@example.com');
    await loginPage.enterPassword('WrongPassword');
    await loginPage.clickSubmit();
    await loginPage.expectErrorMessage('Invalid email or password');
  });

  test('login with empty fields shows validation errors', {
    tag: ['@regression', '@negative'],
  }, async ({ loginPage }) => {
    await loginPage.clickSubmit();
    await loginPage.expectEmailValidationError('Email is required');
    await loginPage.expectPasswordValidationError('Password is required');
  });

});
```

---

### Data-Driven Spec with `test.each()`

```typescript
// tests/authentication/login-validation.spec.ts
import { test, expect } from '../../src/fixtures/test-fixtures';

const invalidEmailCases = [
  { email: 'notanemail',      expectedError: 'Please enter a valid email' },
  { email: '@nodomain.com',   expectedError: 'Please enter a valid email' },
  { email: 'spaces in@email', expectedError: 'Please enter a valid email' },
  { email: '',                expectedError: 'Email is required'          },
];

test.describe('Login — Email Validation', () => {

  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
  });

  for (const { email, expectedError } of invalidEmailCases) {
    test(`email "${email}" shows error: ${expectedError}`, {
      tag: ['@regression', '@negative'],
    }, async ({ loginPage }) => {
      await loginPage.enterEmail(email);
      await loginPage.enterPassword('ValidPass123!');
      await loginPage.clickSubmit();
      await loginPage.expectErrorMessage(expectedError);
    });
  }

});
```

---

### Spec with API Setup (Skip UI Login)

```typescript
// tests/dashboard/dashboard.spec.ts
import { test, expect } from '../../src/fixtures/test-fixtures';
import { ApiHelper } from '../../src/utils/ApiHelper';

test.describe('Dashboard', () => {

  test.beforeEach(async ({ page, context, dashboardPage }) => {
    // Use API to authenticate — skip the login UI for speed
    const api = new ApiHelper(page.request);
    const token = await api.authenticate(
      process.env.TEST_USER_EMAIL!,
      process.env.TEST_USER_PASSWORD!
    );
    // Inject the token as a cookie so the app recognizes the session
    await context.addCookies([{
      name: 'auth_token',
      value: token,
      domain: 'localhost',
      path: '/',
    }]);
    await dashboardPage.goto();
  });

  test('displays personalized welcome message', {
    tag: ['@smoke'],
  }, async ({ dashboardPage }) => {
    await dashboardPage.expectWelcomeMessage(process.env.TEST_USER_EMAIL!);
  });

  test('shows correct statistics panel', {
    tag: ['@regression'],
  }, async ({ dashboardPage }) => {
    await dashboardPage.expectStatsPanelVisible();
  });

});
```

---

### Spec with Shared State via test.use()

```typescript
// tests/admin/admin.spec.ts
import { test, expect } from '../../src/fixtures/test-fixtures';

// All tests in this file share a storage state with admin session
test.use({ storageState: 'test-data/admin-state.json' });

test.describe('Admin Panel', () => {

  test('can access user management', { tag: ['@smoke'] }, async ({ adminPage }) => {
    await adminPage.goto();
    await adminPage.expectUserManagementVisible();
  });

  test('can deactivate a user account', { tag: ['@regression'] }, async ({ adminPage }) => {
    await adminPage.goto();
    await adminPage.deactivateUser('testuser@example.com');
    await adminPage.expectUserDeactivated('testuser@example.com');
  });

});
```

---

## Custom Fixtures Pattern

```typescript
// src/fixtures/test-fixtures.ts
import { test as base, BrowserContext, Page } from '@playwright/test';
import { LoginPage } from '@pages/LoginPage';
import { DashboardPage } from '@pages/DashboardPage';
import { ApiHelper } from '@utils/ApiHelper';
import { LocatorRegistry } from '@utils/LocatorRegistry';

export type TestFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  apiHelper: ApiHelper;
  authenticatedPage: Page;   // page pre-loaded with a valid session
};

export const test = base.extend<TestFixtures>({

  // Flush locator registry after every test
  page: async ({ page }, use) => {
    await use(page);
    await LocatorRegistry.getInstance().save();
  },

  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },

  apiHelper: async ({ page }, use) => {
    await use(new ApiHelper(page.request));
  },

  // A page fixture that is already authenticated via API
  authenticatedPage: async ({ page, context }, use) => {
    const api = new ApiHelper(page.request);
    const token = await api.authenticate(
      process.env.TEST_USER_EMAIL!,
      process.env.TEST_USER_PASSWORD!
    );
    await context.addCookies([{
      name: 'auth_token',
      value: token,
      domain: new URL(process.env.BASE_URL || 'http://localhost:3000').hostname,
      path: '/',
    }]);
    await use(page);
  },

});

export { expect } from '@playwright/test';
```

---

## Step Definition Patterns (playwright-bdd)

Step definitions bind Gherkin steps to Page Object methods using `createBdd(test)`. Always import `test` from `src/fixtures/test-fixtures` — never from `@playwright/test` directly.

### Standard Step Definition File

```typescript
// step-definitions/authentication/login.steps.ts
import { createBdd } from 'playwright-bdd';
import { test } from '../../src/fixtures/test-fixtures';

const { Given, When, Then } = createBdd(test);

// Background steps — shared by all scenarios in this feature
Given('the user is on the login page', async ({ loginPage }) => {
  await loginPage.goto();
});

// Positive flow
When('the user logs in with valid credentials', async ({ loginPage }) => {
  await loginPage.loginWith(
    process.env.TEST_USER_EMAIL!,
    process.env.TEST_USER_PASSWORD!
  );
});

Then('the user should be redirected to the dashboard', async ({ loginPage }) => {
  await loginPage.expectRedirectedToDashboard();
});

// Negative flow — using Cucumber expression {string} for parameterized steps
When('the user enters email {string} and password {string}',
  async ({ loginPage }, email: string, password: string) => {
    await loginPage.enterEmail(email);
    await loginPage.enterPassword(password);
    await loginPage.clickSubmit();
  }
);

Then('an error message {string} should be displayed', async ({ loginPage }, message: string) => {
  await loginPage.expectErrorMessage(message);
});
```

### Step Definitions for Scenario Outline

The feature file:
```gherkin
Scenario Outline: Login validation
  Given the user is on the login page
  When the user attempts login with email "<email>" and password "<password>"
  Then an error message "<error>" should be displayed

  Examples:
    | email             | password   | error                     |
    | wrong@example.com | correct123 | Invalid email or password |
    | user@example.com  | wrongpass  | Invalid email or password |
    |                   |            | Email is required         |
```

The step definition (parameters are injected automatically):
```typescript
When('the user attempts login with email {string} and password {string}',
  async ({ loginPage }, email: string, password: string) => {
    await loginPage.enterEmail(email);
    await loginPage.enterPassword(password);
    await loginPage.clickSubmit();
  }
);
```

### Step Definitions with API Setup

```typescript
// step-definitions/dashboard/dashboard.steps.ts
import { createBdd } from 'playwright-bdd';
import { test } from '../../src/fixtures/test-fixtures';
import { ApiHelper } from '../../src/utils/ApiHelper';

const { Given, When, Then } = createBdd(test);

Given('the user is authenticated', async ({ page, context, dashboardPage }) => {
  const api = new ApiHelper(page.request);
  const token = await api.authenticate(
    process.env.TEST_USER_EMAIL!,
    process.env.TEST_USER_PASSWORD!
  );
  await context.addCookies([{
    name: 'auth_token',
    value: token,
    domain: new URL(process.env.BASE_URL || 'http://localhost:3000').hostname,
    path: '/',
  }]);
  await dashboardPage.goto();
});

Then('the dashboard statistics panel should be visible', async ({ dashboardPage }) => {
  await dashboardPage.expectStatsPanelVisible();
});
```

### Running BDD Tests

```bash
# Step 1 — compile feature files + step definitions into .features-gen/
npx bddgen

# Step 2 — run with Playwright
npx playwright test

# Combined (recommended)
npx bddgen && npx playwright test

# Filter by tag
npx bddgen && npx playwright test --grep "@smoke"

# UI mode for debugging
npx bddgen && npx playwright test --ui
```

---

## Auth State Storage Pattern (Save Once, Reuse)

```typescript
// scripts/save-auth-state.ts — run once before the full suite
import { chromium } from '@playwright/test';
import fs from 'fs';

async function saveAuthState() {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();

  await page.goto(`${process.env.BASE_URL}/login`);
  await page.getByLabel(/email/i).fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel(/password/i).fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: /sign in/i }).click();
  await page.waitForURL(/dashboard/);

  // Save storage state so all tests can reuse it without logging in again
  await context.storageState({ path: 'test-data/user-state.json' });
  await browser.close();
  console.log('✅ Auth state saved to test-data/user-state.json');
}

saveAuthState();
```

Add to `playwright.config.ts`:
```typescript
// globalSetup: './scripts/save-auth-state.ts',
// Or use per-project:
// projects: [{ name: 'chromium', use: { storageState: 'test-data/user-state.json' } }]
```
