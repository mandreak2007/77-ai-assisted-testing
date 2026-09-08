# Code Patterns Reference

Production-ready patterns for POM, step definitions, fixtures, and TypeScript types.

---

## Page Object Model (POM) Patterns

### Standard Page Object

```typescript
// src/pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './BasePage';

export class LoginPage extends BasePage {
  // Locators — private, readonly, defined at class level
  // Priority: data-testid > aria-label/role > placeholder > text > CSS > XPath
  private readonly emailInput: Locator;
  private readonly passwordInput: Locator;
  private readonly submitButton: Locator;
  private readonly errorMessage: Locator;
  private readonly forgotPasswordLink: Locator;
  private readonly rememberMeCheckbox: Locator;

  constructor(page: Page) {
    super(page);
    // Define locators in constructor — evaluated lazily by Playwright
    this.emailInput = page.getByTestId('email-input');
    this.passwordInput = page.getByTestId('password-input');
    this.submitButton = page.getByRole('button', { name: /sign in/i });
    this.errorMessage = page.getByRole('alert');
    this.forgotPasswordLink = page.getByRole('link', { name: /forgot password/i });
    this.rememberMeCheckbox = page.getByRole('checkbox', { name: /remember me/i });
  }

  // Navigation
  async goto(): Promise<void> {
    await this.navigate('/login');
  }

  // Actions — one action per user interaction
  async enterEmail(email: string): Promise<void> {
    await this.emailInput.fill(email);
  }

  async enterPassword(password: string): Promise<void> {
    await this.passwordInput.fill(password);
  }

  async clickSubmit(): Promise<void> {
    await this.submitButton.click();
  }

  async checkRememberMe(): Promise<void> {
    await this.rememberMeCheckbox.check();
  }

  // Compound actions (convenience methods for common flows)
  async loginWith(email: string, password: string): Promise<void> {
    await this.enterEmail(email);
    await this.enterPassword(password);
    await this.clickSubmit();
  }

  // Assertions — always async, always use expect()
  async expectLoginSuccess(): Promise<void> {
    await expect(this.page).toHaveURL(/dashboard/);
  }

  async expectErrorMessage(message: string): Promise<void> {
    await expect(this.errorMessage).toBeVisible();
    await expect(this.errorMessage).toContainText(message);
  }

  async expectEmailFieldInvalid(): Promise<void> {
    await expect(this.emailInput).toHaveAttribute('aria-invalid', 'true');
  }
}
```

---

### Page with Dynamic Content

```typescript
// src/pages/ProductListPage.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './BasePage';

export class ProductListPage extends BasePage {
  private readonly productCards: Locator;
  private readonly searchInput: Locator;
  private readonly sortDropdown: Locator;
  private readonly filterPanel: Locator;
  private readonly loadingSpinner: Locator;
  private readonly paginationNext: Locator;

  constructor(page: Page) {
    super(page);
    this.productCards = page.getByTestId('product-card');
    this.searchInput = page.getByRole('searchbox');
    this.sortDropdown = page.getByRole('combobox', { name: /sort by/i });
    this.filterPanel = page.getByTestId('filter-panel');
    this.loadingSpinner = page.getByTestId('loading-spinner');
    this.paginationNext = page.getByRole('button', { name: /next page/i });
  }

  async goto(): Promise<void> {
    await this.navigate('/products');
  }

  async searchFor(query: string): Promise<void> {
    await this.searchInput.fill(query);
    await this.searchInput.press('Enter');
    // Wait for loading to complete after search
    await this.waitForHidden(this.loadingSpinner);
  }

  async sortBy(option: string): Promise<void> {
    await this.sortDropdown.selectOption({ label: option });
    await this.waitForHidden(this.loadingSpinner);
  }

  async getProductNames(): Promise<string[]> {
    return this.productCards.getByTestId('product-name').allTextContents();
  }

  async getProductCount(): Promise<number> {
    return this.productCards.count();
  }

  async clickProduct(name: string): Promise<void> {
    await this.productCards
      .filter({ hasText: name })
      .first()
      .click();
  }

  async goToNextPage(): Promise<void> {
    await this.paginationNext.click();
    await this.waitForHidden(this.loadingSpinner);
  }

  // Assertions
  async expectProductCount(count: number): Promise<void> {
    await expect(this.productCards).toHaveCount(count);
  }

  async expectProductVisible(name: string): Promise<void> {
    await expect(
      this.productCards.filter({ hasText: name })
    ).toBeVisible();
  }

  async expectSearchResultsFor(query: string): Promise<void> {
    const names = await this.getProductNames();
    const allMatch = names.every(name => 
      name.toLowerCase().includes(query.toLowerCase())
    );
    expect(allMatch, `Expected all results to contain "${query}"`).toBe(true);
  }
}
```

---

## Step Definition Patterns

### Standard Step Definitions

```typescript
// step-definitions/auth/loginSteps.ts
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { LoginPage } from '@pages/LoginPage';
import { CustomWorld } from '@types/world';

// Initialize page objects in Given steps (not globally)
Given('the user is on the login page', async function (this: CustomWorld) {
  this.loginPage = new LoginPage(this.page);
  await this.loginPage.goto();
});

Given('the user is logged in as {string}', async function (
  this: CustomWorld,
  userType: string
) {
  this.loginPage = new LoginPage(this.page);
  await this.loginPage.goto();
  
  const credentials = this.getCredentials(userType);
  await this.loginPage.loginWith(credentials.email, credentials.password);
  await this.loginPage.expectLoginSuccess();
});

When('the user enters email {string}', async function (
  this: CustomWorld,
  email: string
) {
  await this.loginPage.enterEmail(email);
});

When('the user enters password {string}', async function (
  this: CustomWorld,
  password: string
) {
  await this.loginPage.enterPassword(password);
});

When('the user clicks the sign in button', async function (this: CustomWorld) {
  await this.loginPage.clickSubmit();
});

When('the user logs in with valid credentials', async function (this: CustomWorld) {
  await this.loginPage.loginWith(
    process.env.TEST_USER_EMAIL!,
    process.env.TEST_USER_PASSWORD!
  );
});

Then('the user should be redirected to the dashboard', async function (
  this: CustomWorld
) {
  await this.loginPage.expectLoginSuccess();
});

Then('an error message {string} should be displayed', async function (
  this: CustomWorld,
  message: string
) {
  await this.loginPage.expectErrorMessage(message);
});
```

---

### Data Table Steps

```typescript
// Step with Cucumber DataTable
When('the user fills in the registration form:', async function (
  this: CustomWorld,
  dataTable: DataTable
) {
  const formData = dataTable.rowsHash(); // { field: value, ... }
  await this.registrationPage.fillForm(formData);
});

// Step with Cucumber DocString
Given('the following JSON payload:', async function (
  this: CustomWorld,
  docString: string
) {
  this.testData.payload = JSON.parse(docString);
});
```

---

## CustomWorld Extension

```typescript
// src/types/world.ts
import { IWorldOptions, World } from '@cucumber/cucumber';
import { Page, BrowserContext } from '@playwright/test';

// Import all page objects
import { LoginPage } from '@pages/LoginPage';
import { DashboardPage } from '@pages/DashboardPage';

// Credential map type
type UserCredentials = { email: string; password: string };
type UserType = 'standard' | 'admin' | 'readonly';

export class CustomWorld extends World {
  page!: Page;
  context!: BrowserContext;
  scenarioName!: string;

  // Page object instances — initialized in step definitions
  loginPage!: LoginPage;
  dashboardPage!: DashboardPage;

  // Dynamic test data storage
  testData: Record<string, unknown> = {};

  constructor(options: IWorldOptions) {
    super(options);
  }

  /**
   * Get credentials by user type from environment variables
   */
  getCredentials(userType: UserType = 'standard'): UserCredentials {
    const map: Record<UserType, UserCredentials> = {
      standard: {
        email: process.env.TEST_USER_EMAIL!,
        password: process.env.TEST_USER_PASSWORD!,
      },
      admin: {
        email: process.env.ADMIN_USER_EMAIL!,
        password: process.env.ADMIN_USER_PASSWORD!,
      },
      readonly: {
        email: process.env.READONLY_USER_EMAIL!,
        password: process.env.READONLY_USER_PASSWORD!,
      },
    };
    return map[userType];
  }

  /**
   * Store data for later steps
   */
  set<T>(key: string, value: T): void {
    this.testData[key] = value;
  }

  /**
   * Retrieve data from earlier steps
   */
  get<T>(key: string): T {
    return this.testData[key] as T;
  }
}
```

---

## API Setup Pattern (API shortcut for UI state)

```typescript
// src/utils/ApiHelper.ts — Use API calls for test setup, not UI
import { APIRequestContext, expect } from '@playwright/test';

export class ApiHelper {
  constructor(private readonly request: APIRequestContext) {}

  async createUser(userData: {
    email: string;
    password: string;
    role?: string;
  }): Promise<{ id: string; token: string }> {
    const response = await this.request.post('/api/users', {
      data: userData,
    });
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

## Feature File Templates

### Standard Feature

```gherkin
@smoke @auth
Feature: User Authentication
  As a registered user
  I want to securely log in to my account
  So that I can access my personalized content

  Background:
    Given the user is on the login page

  @positive
  Scenario: Successful login with valid credentials
    When the user logs in with valid credentials
    Then the user should be redirected to the dashboard
    And the welcome message should display the user's name

  @negative
  Scenario: Failed login with invalid password
    When the user enters email "user@example.com"
    And the user enters password "WrongPassword"
    And the user clicks the sign in button
    Then an error message "Invalid email or password" should be displayed
    And the user should remain on the login page

  @negative
  Scenario: Login attempt with empty fields
    When the user clicks the sign in button
    Then the email field should show a validation error
    And the password field should show a validation error
```

### Data-Driven Feature

```gherkin
@regression @auth
Feature: Login Validation Rules
  As a security-conscious application
  I want to validate login inputs
  So that only properly formatted credentials are accepted

  @negative
  Scenario Outline: Invalid email format validation
    Given the user is on the login page
    When the user enters email "<email>"
    And the user enters password "ValidPass123!"
    And the user clicks the sign in button
    Then the email field should show the error "<error_message>"

    Examples:
      | email          | error_message                    |
      | notanemail     | Please enter a valid email       |
      | @nodomain.com  | Please enter a valid email       |
      | spaces in@.com | Please enter a valid email       |
      |                | Email is required                |
```
