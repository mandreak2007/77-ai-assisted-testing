# PROJECT_OVERVIEW.pdf Template — MCP (BDD) Mode

Used in Phase 1 (Step 3 — Generate PDF documentation). Replace every `<placeholder>`
with the actual value at generation time (project name from `package.json`, current date, etc.).
Write the filled-in content to `documentation/_tmp_overview.md`, then run:

```bash
pandoc documentation/_tmp_overview.md -o documentation/PROJECT_OVERVIEW.pdf --pdf-engine=wkhtmltopdf
```

Delete `_tmp_overview.md` on success. On failure, leave it in place and report the exact error.

---

# <Project Name> — Playwright Automation Framework

> Created by **Seven Seven Global Inc.**
> Mode: Playwright MCP (BDD) | Generated: <YYYY-MM-DD>

---

## Overview

This project is a **BDD test automation framework** built with Playwright and Cucumber, developed and maintained by **Seven Seven Global Inc.**
It uses the Model Context Protocol (MCP) to inspect live applications and generate self-healing, maintainable test scripts from Gherkin feature files.

---

## Project Folder Structure

```
project-root/
│
├── requirements/               ← INPUT: Drop BRD, user stories, or specs here.
│                                  Claude reads these files in Phase 2 to generate
│                                  feature files and update the master RTM.
│
├── features/                   ← GENERATED: Gherkin .feature files, one per module.
│   └── <module>/               ← Organise by domain (e.g. authentication/, checkout/).
│       └── <module>.feature    ← Contains Scenarios with Given/When/Then steps.
│
├── step-definitions/           ← GENERATED: TypeScript step bindings for Cucumber.
│   └── <module>/               ← Mirrors the features/ structure.
│       └── <module>.steps.ts   ← Each step calls a Page Object method — no locators here.
│
├── src/
│   ├── pages/                  ← Page Object Model (POM) classes.
│   │   ├── BasePage.ts         ← Shared navigation helpers; all pages extend this.
│   │   └── <Name>Page.ts       ← One file per page/component; locators + actions.
│   ├── utils/
│   │   ├── SelfHealingLocator.ts  ← Tries 6 locator strategies; auto-promotes winners.
│   │   └── LocatorRegistry.ts     ← Persists healed strategies to reports/locator-registry.json.
│   ├── hooks/
│   │   └── hooks.ts            ← Cucumber Before/After hooks; browser lifecycle.
│   ├── fixtures/
│   │   └── customFixtures.ts   ← Shared Cucumber fixtures.
│   └── types/
│       └── world.ts            ← CustomWorld interface for cross-step state.
│
├── test-data/                  ← Static test data files (JSON, CSV).
│
├── documentation/              ← Project documentation (this file and any additions).
│   └── PROJECT_OVERVIEW.pdf
│
├── reports/
│   ├── locator-registry.json   ← Live record of all healed locators.
│   └── master-rtm.xlsx         ← Requirements Traceability Matrix (all sessions).
│
├── .github/
│   └── workflows/
│       └── playwright.yml      ← GitHub Actions CI/CD pipeline.
│
├── playwright.config.ts        ← Playwright browser and timeout configuration.
├── cucumber.js                 ← Cucumber runner configuration.
├── tsconfig.json               ← TypeScript compiler settings.
├── package.json                ← Dependencies and npm scripts.
├── .env.example                ← Environment variable template (copy to .env).
└── .gitignore
```

---

## Key Concepts

| Concept | Description |
|---|---|
| **SelfHealingLocator** | Wraps every element with 6 fallback strategies (data-testid → aria → placeholder → text → CSS → XPath). If the primary locator breaks, it tries the next strategy automatically and updates the registry. |
| **LocatorRegistry** | Persists the winning strategy for each element to `reports/locator-registry.json`. Promoted strategies become the new primary on the next run. |
| **CustomWorld** | Holds shared state (page, browser context, logged-in user) across Cucumber steps within a scenario. Never use global variables. |
| **MCP Agent** | The `@playwright/mcp` server lets Claude inspect the live application DOM before generating any code, ensuring all locators are real and verified. |

---

## Getting Started

1. Copy `.env.example` to `.env` and fill in your `BASE_URL` and test credentials.
2. Drop a BRD or user story file into `requirements/`.
3. Tell Claude: *"I added a requirements file to requirements/"*
4. Claude will analyse it (Phase 2) and generate a `.feature` file + update the master RTM.
5. Tell Claude: *"Generate scripts"* to produce Page Objects and step definitions (Phase 3).
6. Tell Claude: *"Run tests"* to execute and self-heal (Phase 4).

---

## Running Tests

```bash
# Run everything
npx cucumber-js

# Run a specific feature file
npx cucumber-js features/<module>/<module>.feature

# Run by tag
npx cucumber-js --tags "@smoke"
npx cucumber-js --tags "@regression"

# Run a single scenario by name
npx cucumber-js --name "<Scenario Name>"
```

---

*This automation framework was created by **Seven Seven Global Inc.***
*For support, contact your Seven Seven Global project team.*
