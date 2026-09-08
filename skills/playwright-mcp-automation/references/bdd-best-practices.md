# BDD Best Practices Reference

Guidelines for writing professional Gherkin feature files and structuring BDD test suites.

---

## Gherkin Writing Principles

### Business Language, Not UI Actions

```gherkin
# BAD — describes UI mechanics, not business intent
Scenario: Login test
  Given I open the browser
  And I navigate to https://app.example.com/login
  When I click on the email field
  And I type "user@example.com" in the email field
  And I click on the password field  
  And I type "Password123" in the password field
  And I click the button with id="submit-btn"
  Then I should see text "Welcome back"

# GOOD — describes business behavior
Scenario: Registered user accesses their account
  Given a registered user is on the login page
  When the user signs in with valid credentials
  Then the user should be on their dashboard
  And a personalized welcome message should appear
```

---

## Tag Taxonomy

Define a consistent tagging strategy across all features:

| Tag | Usage |
|---|---|
| `@smoke` | Critical path — run on every deploy (~5-10 mins) |
| `@regression` | Full suite — run nightly or before release |
| `@wip` | Work in progress — excluded from CI by default |
| `@positive` | Happy path scenarios |
| `@negative` | Error/edge case scenarios |
| `@api` | API-layer tests |
| `@ui` | Browser UI tests |
| `@accessibility` | A11y compliance tests |
| `@performance` | Load/timing tests |
| `@mobile` | Mobile viewport scenarios |
| `@data-driven` | Scenario Outlines with Examples |
| `@<domain>` | Domain-specific: `@auth`, `@checkout`, `@search` |

---

## Scenario Writing Rules

1. **One behavior per scenario** — test one thing, not a whole workflow
2. **Given = Context** (system state, not past actions)
3. **When = Action** (exactly one key action per scenario)
4. **Then = Observable outcome** (what can the user see/verify?)
5. **And/But** = continuation of same keyword — never mix contexts
6. **Max 7 steps** — if more, split the scenario or add a Background
7. **No "I"** — use third-person: "the user", "the admin", "the customer"
8. **No implementation details** — no selectors, IDs, CSS classes

---

## Background Usage

```gherkin
# Use Background for preconditions shared by ALL scenarios in a feature
# If only SOME scenarios share it — use a step instead

Feature: Shopping Cart
  Background:
    Given a logged-in customer with items in their cart
    # This applies to every scenario below — appropriate use

  Scenario: Customer views cart total
    ...
  
  Scenario: Customer removes an item
    ...
```

---

## Scenario Outline Best Practices

```gherkin
# Good — meaningful column names, realistic data, covers edge cases
Scenario Outline: Password strength validation
  Given the user is on the registration page
  When the user enters password "<password>"
  Then the strength indicator should show "<strength>"
  And the submit button should be "<button_state>"

  Examples:
    | password          | strength | button_state |
    | abc               | Weak     | disabled     |
    | password123       | Fair     | disabled     |
    | P@ssw0rd          | Strong   | enabled      |
    | C0mpl3x!P@ss#2024 | Very Strong | enabled   |

# Avoid — single-row Outlines (just use a Scenario)
# Avoid — more than ~8-10 example rows (split into multiple features)
```

---

## Feature File Structure

```gherkin
@domain-tag @test-type
Feature: [Business Capability Name]
  As a [role]
  I want [goal]
  So that [benefit]

  # Optional: shared preconditions for ALL scenarios
  Background:
    Given [shared precondition]

  # Positive/happy path scenarios first
  @smoke @positive
  Scenario: [Most important happy path]
    Given ...
    When ...
    Then ...

  # Secondary positive paths
  @regression @positive  
  Scenario: [Secondary happy path]
    ...

  # Negative/edge cases
  @regression @negative
  Scenario: [Error case]
    ...

  # Data-driven scenarios last
  @regression @data-driven
  Scenario Outline: [Multiple variations]
    ...
    Examples:
      ...
```

---

## Step Definition Reusability

```typescript
// GOOD — Generic, reusable steps that work across features
Given('the user is on the {string} page', async function (this: CustomWorld, pageName: string) {
  const pageMap: Record<string, string> = {
    'login': '/login',
    'registration': '/register',
    'dashboard': '/dashboard',
    'profile': '/profile/settings',
  };
  await this.page.goto(`${process.env.BASE_URL}${pageMap[pageName]}`);
});

// AVOID — Steps so specific they can only be used in one scenario
Given('the user navigates to the login page and waits for the form to load', ...)
```

---

## Test Data Strategy

### Externalize in JSON

```json
// test-data/users.json
{
  "standard": {
    "email": "standard@test.com",
    "password": "Password123!",
    "role": "USER"
  },
  "admin": {
    "email": "admin@test.com",
    "password": "AdminPass123!",
    "role": "ADMIN"
  }
}
```

### Use Scenario Context for Dynamic Data

```typescript
// Store generated data in World for cross-step sharing
When('a new user registers', async function (this: CustomWorld) {
  const email = `test-${Date.now()}@example.com`;
  this.set('registeredEmail', email);
  await this.registrationPage.register({ email, password: 'Password123!' });
});

Then('the user can log in with the registered email', async function (this: CustomWorld) {
  const email = this.get<string>('registeredEmail');
  await this.loginPage.loginWith(email, 'Password123!');
});
```

---

## Folder Naming Convention

```
features/
├── authentication/
│   ├── login.feature
│   ├── registration.feature
│   └── password-reset.feature
├── shopping-cart/
│   ├── add-to-cart.feature
│   ├── checkout.feature
│   └── order-confirmation.feature
├── user-profile/
│   └── profile-management.feature
└── search/
    └── product-search.feature

step-definitions/
├── authentication/
│   ├── loginSteps.ts
│   └── registrationSteps.ts
├── shopping-cart/
│   └── cartSteps.ts
└── common/
    └── commonSteps.ts     ← Shared steps across all domains
```
