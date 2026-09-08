# BDD Patterns for Playwright CLI

The CLI automation mode uses `playwright-bdd` for full Gherkin feature file support while keeping the native Playwright test runner. A secondary approach using native `test.describe()` is available for teams that prefer pure TypeScript without feature files.

---

## Approach 1 — `playwright-bdd` Package (Full Gherkin Support) — Default

For full `.feature` files with Cucumber Gherkin syntax, use `playwright-bdd`. This keeps the Playwright test runner while adding Gherkin parsing.

### Install

```bash
npm install -D playwright-bdd
```

### `playwright.config.ts` with BDD

```typescript
import { defineConfig } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';

const testDir = defineBddConfig({
  features: 'features/**/*.feature',
  steps: 'step-definitions/**/*.ts',
});

export default defineConfig({
  testDir,
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
  },
});
```

### Feature File (identical to Cucumber format)

```gherkin
# features/authentication/login.feature
@smoke @auth
Feature: User Authentication
  As a registered user
  I want to securely log in
  So that I can access my account

  Background:
    Given the user is on the login page

  @positive
  Scenario: Successful login with valid credentials
    When the user logs in with valid credentials
    Then the user should be redirected to the dashboard

  @negative
  Scenario: Failed login with invalid password
    When the user enters email "user@example.com"
    And the user enters password "WrongPassword"
    And the user clicks sign in
    Then an error message "Invalid email or password" should be displayed
```

### Step Definitions (playwright-bdd style)

```typescript
// step-definitions/auth/loginSteps.ts
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

When('the user enters email {string}', async ({ loginPage }, email: string) => {
  await loginPage.enterEmail(email);
});

When('the user enters password {string}', async ({ loginPage }, password: string) => {
  await loginPage.enterPassword(password);
});

When('the user clicks sign in', async ({ loginPage }) => {
  await loginPage.clickSubmit();
});

Then('the user should be redirected to the dashboard', async ({ loginPage }) => {
  await loginPage.expectRedirectedToDashboard();
});

Then('an error message {string} should be displayed', async ({ loginPage }, message: string) => {
  await loginPage.expectErrorMessage(message);
});
```

### Running playwright-bdd Tests

```bash
# Generate test files from feature files (required step)
npx bddgen

# Then run normally with Playwright CLI
npx playwright test

# Or combine into one command
npx bddgen && npx playwright test
```

Add to `package.json`:
```json
{
  "scripts": {
    "test": "bddgen && playwright test",
    "test:smoke": "bddgen && playwright test --grep @smoke"
  }
}
```

---

## Approach 2 — Native BDD Style (No Extra Packages)

Use `test.describe()` + `test()` with BDD naming conventions. No Cucumber required.

### Naming Convention

```typescript
// tests/authentication/login.spec.ts

test.describe('Authentication', () => {
  // "Feature: Authentication"

  test.describe('Successful login', () => {
    // "Scenario: Successful login with valid credentials"

    test.beforeEach(async ({ loginPage }) => {
      // "Given the user is on the login page"
      await loginPage.goto();
    });

    test('redirects to the dashboard', {
      tag: ['@smoke', '@positive'],
      // "Then the user should be on the dashboard"
    }, async ({ loginPage }) => {
      // "When the user logs in with valid credentials"
      await loginPage.loginWith(
        process.env.TEST_USER_EMAIL!,
        process.env.TEST_USER_PASSWORD!
      );

      // "Then the user should be redirected to the dashboard"
      await loginPage.expectRedirectedToDashboard();
    });
  });

  test.describe('Failed login', () => {
    // "Scenario: Failed login with invalid credentials"

    test.beforeEach(async ({ loginPage }) => {
      await loginPage.goto();
    });

    test('shows error for wrong password', {
      tag: ['@regression', '@negative'],
    }, async ({ loginPage }) => {
      await loginPage.enterEmail('user@example.com');
      await loginPage.enterPassword('WrongPassword');
      await loginPage.clickSubmit();
      await loginPage.expectErrorMessage('Invalid email or password');
    });
  });
});
```

### Why This Works Well

- `test.describe()` = Feature / Scenario group
- `test()` = Scenario / test case
- `test.beforeEach()` = Background / Given steps
- Tags applied in `test()` options replace Gherkin `@tags`
- No extra tooling, no process boundary between BDD and runner

---

## Choosing Between Approaches

| Factor | playwright-bdd (default) | Native test.describe() |
|---|---|---|
| Extra package | `playwright-bdd` | None |
| Feature files | Yes — `.feature` + Gherkin | No — TypeScript only |
| BDD stakeholder review | Easy — read `.feature` files | Harder |
| Step reuse | Via step definition files | Via shared helpers |
| Setup complexity | Medium | Minimal |
| Debuggability | Excellent (Playwright UI mode works) | Excellent |
| CI/CD | Add `bddgen` step before `playwright test` | Standard |
| Recommended for | Default — BDD traceability + Playwright tooling | Dev teams wanting pure TypeScript |

---

## Test Tagging Strategy (Both Approaches)

Consistent tag taxonomy for filtering:

| Tag | Meaning | Run frequency |
|---|---|---|
| `@smoke` | Critical path — fast subset | Every deploy |
| `@regression` | Full suite | Nightly / pre-release |
| `@positive` | Happy path tests | Included in smoke + regression |
| `@negative` | Error / edge cases | Regression only |
| `@api` | API-layer tests | With regression |
| `@ui` | Browser UI tests | With smoke + regression |
| `@auth` | Authentication domain | Smoke subset |
| `@checkout` | Checkout domain | Regression subset |
| `@wip` | Work in progress | Never in CI |

### Applying Tags in Native Playwright

```typescript
test('some test', { tag: ['@smoke', '@positive'] }, async ({ page }) => {
  // ...
});

// Or at describe level — applies to all tests inside
test.describe('Login flows', { tag: ['@auth'] }, () => {
  test('...', { tag: ['@smoke'] }, async () => { /* ... */ });
  test('...', { tag: ['@regression'] }, async () => { /* ... */ });
});
```

### Running by Tag

```bash
npx playwright test --grep "@smoke"
npx playwright test --grep "@regression"
npx playwright test --grep "@auth"
npx playwright test --grep-invert "@wip"    # exclude WIP tests
```

---

## Data-Driven Tests

### Native Playwright — test.each()-style Loop

```typescript
const loginFailureCases = [
  { email: 'wrong@email.com', password: 'correct', expectedError: 'Invalid email or password' },
  { email: 'user@example.com', password: 'wrongpass', expectedError: 'Invalid email or password' },
  { email: '',                 password: '',          expectedError: 'Email is required'          },
];

test.describe('Login — negative cases', () => {
  for (const { email, password, expectedError } of loginFailureCases) {
    test(`login fails — email="${email}"`, {
      tag: ['@regression', '@negative'],
    }, async ({ loginPage }) => {
      await loginPage.goto();
      await loginPage.enterEmail(email);
      await loginPage.enterPassword(password);
      await loginPage.clickSubmit();
      await loginPage.expectErrorMessage(expectedError);
    });
  }
});
```

### playwright-bdd — Scenario Outline

```gherkin
@regression @negative @data-driven
Scenario Outline: Login validation
  Given the user is on the login page
  When the user enters email "<email>"
  And the user enters password "<password>"
  And the user clicks sign in
  Then an error message "<error>" should be displayed

  Examples:
    | email            | password  | error                     |
    | wrong@email.com  | correct   | Invalid email or password |
    | user@example.com | wrongpass | Invalid email or password |
    |                  |           | Email is required         |
```

---

## Folder Structure Comparison

### Native test.describe() structure

```
tests/
├── authentication/
│   ├── login.spec.ts
│   └── registration.spec.ts
├── checkout/
│   └── checkout.spec.ts
└── common/
    └── navigation.spec.ts

src/pages/
├── BasePage.ts
├── LoginPage.ts
└── CheckoutPage.ts

src/fixtures/
└── test-fixtures.ts
```

### playwright-bdd structure

```
features/
├── authentication/
│   └── login.feature
└── checkout/
    └── checkout.feature

step-definitions/
├── authentication/
│   └── loginSteps.ts
└── checkout/
    └── checkoutSteps.ts

src/pages/          (identical — POM is shared)
src/fixtures/       (identical — fixtures are shared)

.features-gen/      (auto-generated — don't edit)
```
