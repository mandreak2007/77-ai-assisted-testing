# GitLab CI — Pipeline Templates

Use these templates when the user selects GitLab CI (option 2).
Output file: `.gitlab-ci.yml`

Always replace every `<placeholder>` with a real value. Never leave angle-bracket placeholders in the final file.

---

## CLI Mode Template

Use when the project runner is `npx playwright test`.

```yaml
image: mcr.microsoft.com/playwright:v1.50.0-noble

stages:
  - test
  - report

variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"
  PLAYWRIGHT_BROWSERS_PATH: "$CI_PROJECT_DIR/.playwright-browsers"

cache:
  key:
    files:
      - package-lock.json
  paths:
    - .npm/
    - .playwright-browsers/

.test-base:
  stage: test
  before_script:
    - npm ci --cache .npm --prefer-offline
    - npx playwright install chromium
  artifacts:
    when: always
    paths:
      - blob-report/
    expire_in: 1 day

test-shard-1:
  extends: .test-base
  script:
    - npx playwright test --shard=1/4
  variables:
    BASE_URL: $BASE_URL
    TEST_USER_EMAIL: $TEST_USER_EMAIL
    TEST_USER_PASSWORD: $TEST_USER_PASSWORD

test-shard-2:
  extends: .test-base
  script:
    - npx playwright test --shard=2/4

test-shard-3:
  extends: .test-base
  script:
    - npx playwright test --shard=3/4

test-shard-4:
  extends: .test-base
  script:
    - npx playwright test --shard=4/4

merge-reports:
  stage: report
  when: always
  needs:
    - job: test-shard-1
      artifacts: true
    - job: test-shard-2
      artifacts: true
    - job: test-shard-3
      artifacts: true
    - job: test-shard-4
      artifacts: true
  before_script:
    - npm ci --cache .npm --prefer-offline
  script:
    - npx playwright merge-reports --reporter html ./blob-report
  artifacts:
    when: always
    paths:
      - playwright-report/
    expire_in: 30 days
    expose_as: "Playwright HTML Report"
```

---

## MCP (BDD) Mode Template

Use when the project runner is `npx cucumber-js`.

```yaml
image: mcr.microsoft.com/playwright:v1.50.0-noble

stages:
  - test

variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"
  PLAYWRIGHT_BROWSERS_PATH: "$CI_PROJECT_DIR/.playwright-browsers"

cache:
  key:
    files:
      - package-lock.json
  paths:
    - .npm/
    - .playwright-browsers/

bdd-tests:
  stage: test
  before_script:
    - npm ci --cache .npm --prefer-offline
    - npx playwright install chromium
  script:
    - npx cucumber-js
  variables:
    BASE_URL: $BASE_URL
    TEST_USER_EMAIL: $TEST_USER_EMAIL
    TEST_USER_PASSWORD: $TEST_USER_PASSWORD
  artifacts:
    when: always
    paths:
      - reports/
    expire_in: 30 days
    expose_as: "Cucumber Report"
```

---

## Notes

- The `mcr.microsoft.com/playwright` Docker image already contains all browser system dependencies — no `apt-get` needed.
- Update the image tag (`v1.50.0-noble`) to match the version in your `package.json` (`@playwright/test` version).
- Add environment variables under **Settings → CI/CD → Variables** in GitLab. Mark sensitive ones as **Masked**.
- `expose_as` makes the artifact downloadable directly from the pipeline UI.
- For GitLab self-managed, ensure your runner has Docker executor enabled.
