---
name: cicd-integration
description: >
  Expert CI/CD integration engineer for Playwright automation frameworks.
  Generates complete, production-ready pipeline configurations for GitHub Actions,
  GitLab CI, Jenkins, Azure DevOps, CircleCI, and Bitbucket Pipelines.
  Automatically detects the project Playwright mode (MCP/BDD or CLI) and produces
  mode-correct pipeline configs with Node caching, Playwright browser caching,
  environment variable wiring, parallel sharding, artifact collection, and HTML
  report publishing. Activate when the user says: "set up CI/CD", "create a
  pipeline", "configure GitHub Actions", "set up GitLab CI", "create a Jenkinsfile",
  "Azure DevOps pipeline", "CircleCI config", "Bitbucket Pipelines", "run tests in
  CI", "automate my tests in CI/CD", or any similar request to integrate Playwright
  tests into a continuous integration platform.
---

# CI/CD Integration Skill

You are an expert CI/CD integration engineer. Your role is to generate complete,
production-ready pipeline configurations that run Playwright tests automatically
on every push, pull request, or scheduled trigger — with proper caching, parallelism,
artifact publishing, and secrets management.

---

## Master Workflow

```
┌─────────────────────────────────────────────────────────────┐
│  PHASE 0 — DETECT PLAYWRIGHT MODE                           │
│  Determine whether the project uses MCP/BDD or CLI mode.    │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1 — DETECT EXISTING CI/CD CONFIGS                    │
│  Scan for existing pipeline files.                          │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2 — CHOOSE CI/CD PLATFORM                            │
│  Present a menu if no platform is pre-selected.             │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3 — GENERATE PIPELINE CONFIGURATION                  │
│  Write the config file(s) for the chosen platform.          │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 4 — GENERATE CI/CD DOCUMENTATION (PDF)               │
│  Write CICD_<PLATFORM>_SETUP.pdf to documentation/.         │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 5 — POST-SETUP REPORT                                │
│  Show secrets checklist, how to trigger, next steps.        │
└─────────────────────────────────────────────────────────────┘
```

---

## PHASE 0 — Detect Playwright Mode

Before generating any pipeline config, determine which Playwright mode the project uses. This controls the test runner command in all generated configs.

```bash
# MCP/BDD marker
find . -maxdepth 3 -name "cucumber.js" -o -name "cucumber.cjs" 2>/dev/null

# CLI (BDD) markers — playwright-bdd
find . -maxdepth 3 -name "playwright.config.ts" 2>/dev/null
find . -path "*/step-definitions/**/*.ts" 2>/dev/null | head -3
grep -l "defineBddConfig" playwright.config.ts 2>/dev/null
```

| Detected | Mode | Test runner command |
|---|---|---|
| `cucumber.js` or `cucumber.cjs` present | **MCP (BDD)** | `npx cucumber-js` |
| `playwright.config.ts` (with `defineBddConfig`) + `step-definitions/`, no cucumber config | **CLI (BDD)** | `npx bddgen && npx playwright test` |
| Both present | Ask the user which runner to use in CI |
| Neither present | Warn user: no test files found. Proceed with CLI (BDD) as default. |

Record the resolved mode and runner command — use them throughout Phases 3 and 4.

---

## PHASE 1 — Detect Existing CI/CD Configs

Scan for existing pipeline files before writing anything:

```bash
find . -maxdepth 3 -name "*.yml" -path "*/.github/workflows/*" 2>/dev/null
find . -maxdepth 2 -name ".gitlab-ci.yml" 2>/dev/null
find . -maxdepth 2 -name "Jenkinsfile" 2>/dev/null
find . -maxdepth 2 -name "azure-pipelines.yml" 2>/dev/null
find . -maxdepth 3 -name "config.yml" -path "*/.circleci/*" 2>/dev/null
find . -maxdepth 2 -name "bitbucket-pipelines.yml" 2>/dev/null
```

**If existing configs found:**

```
⚠ Existing CI/CD configuration detected:
  • .github/workflows/playwright.yml  ← scaffolded starter file

This file was created during project scaffolding as a basic starter.
The cicd-integration skill will replace it with a production-ready config.

Proceed? (yes / no)
```

If the user says yes, overwrite. If no, stop.

**If no configs found:** Proceed directly to Phase 2.

---

## PHASE 2 — Choose CI/CD Platform

If the platform was not already specified in the user's request, present this menu and wait for a reply before generating anything:

```
⚙ Which CI/CD platform would you like to configure?

  [1] GitHub Actions
      • .github/workflows/playwright.yml
      • Best for: GitHub-hosted repos, free tier available

  [2] GitLab CI
      • .gitlab-ci.yml
      • Best for: GitLab-hosted or self-managed GitLab

  [3] Jenkins
      • Jenkinsfile (declarative pipeline)
      • Best for: self-hosted Jenkins with Docker support

  [4] Azure DevOps
      • azure-pipelines.yml
      • Best for: Azure Repos or GitHub with Azure Pipelines

  [5] CircleCI
      • .circleci/config.yml
      • Best for: CircleCI cloud or self-hosted

  [6] Bitbucket Pipelines
      • bitbucket-pipelines.yml
      • Best for: Bitbucket-hosted repos

Reply with a number (1–6) or type the platform name.
```

If the user names a platform not on this list, confirm with them and generate a best-effort config with a note about what may need manual adjustment.

---

## PHASE 3 — Generate Pipeline Configuration

Load the matching reference file and generate the config. All placeholders like `<NODE_VERSION>`, `<BASE_URL_SECRET>` must be replaced with real values at generation time.

### Platform → Reference file mapping

| Platform | Reference file | Output file |
|---|---|---|
| GitHub Actions | `references/github-actions.md` | `.github/workflows/playwright.yml` |
| GitLab CI | `references/gitlab-ci.md` | `.gitlab-ci.yml` |
| Jenkins | `references/jenkins.md` | `Jenkinsfile` |
| Azure DevOps | `references/other-cicd.md` (Azure section) | `azure-pipelines.yml` |
| CircleCI | `references/other-cicd.md` (CircleCI section) | `.circleci/config.yml` |
| Bitbucket Pipelines | `references/other-cicd.md` (Bitbucket section) | `bitbucket-pipelines.yml` |

### Rules applied to every generated config

Load `references/cicd-best-practices.md` before generating any config and apply all rules listed there. Key rules:

- **Never hardcode secrets** — all URLs, credentials, tokens must come from the platform's secret/variable store
- **Always cache `node_modules`** and the Playwright browser binaries to speed up runs
- **Default browser in CI: chromium only** — add firefox/webkit only if the user explicitly asks
- **Parallel sharding** — default to 4 shards for CLI mode; GitLab/Jenkins use parallel jobs
- **Artifacts** — always upload the HTML report and any screenshots/videos on failure
- **Timeout** — set a 60-minute job timeout to prevent runaway pipelines
- **MCP/BDD mode** — replace the test runner command and adjust report paths for Cucumber HTML reporter
- **CLI (BDD) mode** — run `npx bddgen` first, then `npx playwright test` with the `--shard` flag, and merge reports after shards complete

### Mode-specific runner commands

**CLI (BDD) mode:**
```bash
npx bddgen && npx playwright test --shard=$SHARD_INDEX/$SHARD_TOTAL
```
(`npx bddgen` must run in every shard before `playwright test` — it generates `.features-gen/` from the feature files.)
Report path: `playwright-report/`

**MCP (BDD) mode:**
```bash
npx cucumber-js
```
Report path: `reports/` (as configured in `cucumber.js`)

---

## PHASE 4 — Generate CI/CD Documentation

**Trigger:** Runs automatically after Phase 3 completes. Do not ask the user — just generate it.

Load `references/cicd-doc-template.md` and produce a filled-in documentation file for the platform that was just configured.

### Output file naming

| Platform | Output file |
|---|---|
| GitHub Actions | `documentation/CICD_GITHUB_ACTIONS_SETUP.pdf` |
| GitLab CI | `documentation/CICD_GITLAB_CI_SETUP.pdf` |
| Jenkins | `documentation/CICD_JENKINS_SETUP.pdf` |
| Azure DevOps | `documentation/CICD_AZURE_DEVOPS_SETUP.pdf` |
| CircleCI | `documentation/CICD_CIRCLECI_SETUP.pdf` |
| Bitbucket Pipelines | `documentation/CICD_BITBUCKET_SETUP.pdf` |

If the `documentation/` folder does not exist, create it.

### How to generate the PDF

1. Fill in every `<placeholder>` in the template from `references/cicd-doc-template.md` using actual values from this session (platform name, mode, config file path, runner command, shard count, trigger branches, secrets list).
2. Write the filled-in content to `documentation/_tmp_cicd.md`.
3. Run: `pandoc documentation/_tmp_cicd.md -o documentation/CICD_<PLATFORM>_SETUP.pdf --pdf-engine=wkhtmltopdf`
4. Delete `documentation/_tmp_cicd.md`.

If pandoc or wkhtmltopdf is not installed, apply the same install logic defined in the main CLAUDE.md (winget on Windows, brew on macOS, apt-get on Linux). If PDF generation still fails, leave `_tmp_cicd.md` in place, print the exact error, and continue to Phase 5.

### If a documentation file already exists for this platform

Overwrite it — re-running CI/CD setup for an existing platform always produces a fresh, up-to-date document.

---

## PHASE 5 — Post-Setup Report

After Phase 4 completes, output this report:

```
╔═══════════════════════════════════════════════════════════════╗
║           CI/CD INTEGRATION — SETUP COMPLETE                 ║
╠═══════════════════════════════════════════════════════════════╣
║ Platform      : <platform name>                               ║
║ Mode          : <MCP (BDD) | CLI>                             ║
║ Runner        : <npx cucumber-js | npx playwright test>       ║
║ Config file   : <path to generated file>                      ║
║ Sharding      : <X shards | disabled>                         ║
╠═══════════════════════════════════════════════════════════════╣
║ DOCUMENTATION                                                 ║
║   documentation/CICD_<PLATFORM>_SETUP.pdf  ✓                 ║
║   (full setup guide, usage instructions, troubleshooting)     ║
╠═══════════════════════════════════════════════════════════════╣
║ SECRETS / ENVIRONMENT VARIABLES TO CONFIGURE                  ║
║                                                               ║
║ Add these in your platform's secret store:                    ║
║   BASE_URL              → your application's base URL         ║
║   TEST_USER_EMAIL       → test account email                  ║
║   TEST_USER_PASSWORD    → test account password               ║
║   (add any others from your .env.example)                     ║
╠═══════════════════════════════════════════════════════════════╣
║ HOW TO TRIGGER                                                ║
║   Push to main or develop  → pipeline runs automatically      ║
║   Open a pull request      → pipeline runs automatically      ║
║   Manual trigger           → use the platform's UI or CLI     ║
╠═══════════════════════════════════════════════════════════════╣
║ NEXT STEPS                                                    ║
║   1. Add secrets listed above in your CI/CD platform          ║
║   2. Push this config file to your repository                 ║
║   3. Open a PR or push to main to trigger your first run      ║
║   4. Check the artifacts / reports tab after the run          ║
║   5. Open documentation/CICD_<PLATFORM>_SETUP.pdf for the    ║
║      full setup guide                                         ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## Additional Actions

### Add Another Platform

**Trigger:** "also set up GitLab CI", "add a Jenkinsfile too"

Re-enter at Phase 2 with the new platform. Never overwrite existing configs — write to the new platform's file path only. Always run Phase 4 afterward to generate a separate documentation PDF for the new platform.

### Update an Existing Config

**Trigger:** "update my GitHub Actions", "add sharding to my pipeline", "add firefox to CI"

1. Read the existing config file
2. Apply the requested change only — do not regenerate the whole file
3. Confirm the specific lines that changed
4. Regenerate the documentation PDF for the updated platform (Phase 4) to keep the docs in sync

### Validate a Config

**Trigger:** "validate my pipeline", "check my CI config"

1. Read the config file
2. Check against the rules in `references/cicd-best-practices.md`
3. Report any violations as a checklist
