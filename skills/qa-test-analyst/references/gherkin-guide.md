# Gherkin / BDD Feature File Guide

## File Structure

```gherkin
@feature-tag
Feature: <Feature Name>
  As a <role/actor>
  I want to <goal>
  So that <business value>

  Background:
    Given <common preconditions shared by all scenarios in this feature>

  @tc-TC-001 @req-REQ-001 @positive @smoke
  Scenario: TC-001 — <Short descriptive title — what this test proves>
    Given <system/world is in a specific state>
    When  <user/actor performs an action>
    Then  <observable outcome is verified>
    And   <additional assertion if needed>

  @tc-TC-002 @req-REQ-001 @negative
  Scenario: TC-002 — <Title describing the failure case>
    Given ...
    When  ...
    Then  ...

  @tc-TC-003 @req-REQ-002 @positive @data-driven
  Scenario Outline: TC-003 — <Title for data-driven scenario>
    Given <state with <placeholder>>
    When  <action with <placeholder>>
    Then  <outcome with <expected_result>>

    Examples:
      | placeholder | expected_result |
      | value1      | result1         |
      | value2      | result2         |
```

---

## Tagging Strategy

Every scenario must have at minimum:
- `@tc-TC-XXX` — **unit test case number** (e.g. `@tc-TC-001`); must be the **first tag** on the line and match the TC-ID in the scenario title prefix
- `@req-REQ-XXX` — links the scenario to a requirement ID
- One test type tag: `@positive`, `@negative`, `@boundary`, `@business-rule`

### TC-ID in the Scenario Title (mandatory)

Prefix every `Scenario:` and `Scenario Outline:` title with its TC-ID followed by an em dash:

```gherkin
@tc-TC-001 @req-REQ-001 @positive @smoke
Scenario: TC-001 — Successful login with valid credentials
```

This makes the test case number visible in the Gherkin file, test runner output,
and any generated report — without requiring a lookup in the RTM.

Optional tags (add as appropriate):
- `@smoke` — included in smoke test suite
- `@regression` — included in regression suite
- `@critical` — high business impact
- `@data-driven` — uses Scenario Outline/Examples
- `@wip` — work in progress, not yet executable

---

## Step Writing Rules

### Given — Set up the state
- Describe the system/data state BEFORE the action
- Reference specific test data (usernames, amounts, dates)
- ✅ `Given the user "john.doe@example.com" has an active account`
- ❌ `Given a valid user is logged in`

### When — The action
- ONE primary action per scenario
- Be specific about what is clicked, submitted, entered
- ✅ `When the user submits the login form with password "Passw0rd!"`
- ❌ `When the user logs in`

### Then — The observable result
- Verify what the USER can see or what the SYSTEM state becomes
- Use specific values, messages, statuses
- ✅ `Then the user should see the dashboard with greeting "Welcome, John"`
- ❌ `Then the login should be successful`

### And / But
- `And` continues the same keyword (Given, When, or Then)
- `But` adds a contrasting assertion
- Never start a step with `And` as the very first step

---

## Scenario Outline (Data-Driven Tests)

Use `Scenario Outline:` when the same flow needs multiple data combinations.
Always define the `Examples:` table with clear column headers.

```gherkin
@tc-TC-005 @req-REQ-005 @boundary @data-driven
Scenario Outline: TC-005 — Password validation enforces length rules
  Given the registration form is displayed
  When  the user enters password "<password>"
  Then  the system should display "<expected_message>"

  Examples:
    | password      | expected_message                          |
    |               | Password is required                      |
    | abc           | Password must be at least 8 characters    |
    | validPass1!   | Password accepted                         |
    | A1!aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa | Password must not exceed 40 characters |
```

---

## Background Section

Use `Background:` for steps that are identical across ALL scenarios in a feature.
Do not use it for steps that apply to only some scenarios.

```gherkin
Background:
  Given the application is running at "https://app.example.com"
  And   the database has been seeded with test data
```

---

## Naming Conventions

- Feature file name: `kebab-case.feature` (e.g., `user-login.feature`)
- Feature title: Title Case
- Scenario title: Sentence case, describes WHAT is being proven
- Step text: Natural language, present tense

---

## Full Example

```gherkin
@login
Feature: User Login
  As a registered user
  I want to log in to the application
  So that I can access my personalized dashboard

  Background:
    Given the login page is displayed at "/login"

  @tc-TC-001 @req-REQ-001 @positive @smoke
  Scenario: TC-001 — Successful login with valid credentials
    Given the user "john.doe@example.com" has an active account with password "Secure@123"
    When  the user enters email "john.doe@example.com" and password "Secure@123"
    And   the user clicks the "Login" button
    Then  the user should be redirected to the dashboard at "/dashboard"
    And   the welcome message "Welcome, John Doe" should be displayed

  @tc-TC-002 @req-REQ-001 @negative
  Scenario: TC-002 — Login fails with incorrect password
    Given the user "john.doe@example.com" has an active account
    When  the user enters email "john.doe@example.com" and password "WrongPass!"
    And   the user clicks the "Login" button
    Then  the login should be rejected
    And   the error message "Invalid email or password" should be displayed
    And   the user should remain on the login page

  @tc-TC-003 @req-REQ-002 @negative @boundary
  Scenario: TC-003 — Login fails with unregistered email
    Given no account exists for "unknown@example.com"
    When  the user enters email "unknown@example.com" and password "AnyPass@1"
    And   the user clicks the "Login" button
    Then  the error message "Invalid email or password" should be displayed

  @tc-TC-004 @req-REQ-003 @negative @boundary
  Scenario Outline: TC-004 — Login form validates required fields
    Given the login form is displayed
    When  the user enters email "<email>" and password "<password>"
    And   the user clicks the "Login" button
    Then  the validation message "<message>" should be displayed

    Examples:
      | email                   | password   | message                    |
      |                         | Secure@123 | Email is required          |
      | john.doe@example.com    |            | Password is required       |
      |                         |            | Email and password required |
      | not-an-email            | Secure@123 | Enter a valid email address |
```
