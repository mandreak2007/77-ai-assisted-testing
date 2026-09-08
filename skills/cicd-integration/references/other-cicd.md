# Other CI/CD Platforms — Pipeline Templates

Templates for Azure DevOps, CircleCI, and Bitbucket Pipelines.
Each section includes both CLI mode and MCP (BDD) mode variants.

---

## Azure DevOps

Output file: `azure-pipelines.yml`

### CLI Mode

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pr:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  NODE_VERSION: '20'
  npm_config_cache: $(Pipeline.Workspace)/.npm

strategy:
  parallel: 4

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: $(NODE_VERSION)
    displayName: 'Install Node.js'

  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      restoreKeys: |
        npm | "$(Agent.OS)"
      path: $(npm_config_cache)
    displayName: 'Cache npm packages'

  - task: Cache@2
    inputs:
      key: 'playwright | "$(Agent.OS)" | package-lock.json'
      restoreKeys: |
        playwright | "$(Agent.OS)"
      path: $(Pipeline.Workspace)/.playwright-browsers
    displayName: 'Cache Playwright browsers'

  - script: npm ci --cache $(npm_config_cache) --prefer-offline
    displayName: 'Install dependencies'

  - script: npx playwright install chromium
    displayName: 'Install Playwright browsers'

  - script: npx playwright test --shard=$(System.JobPositionInPhase)/$(System.TotalJobsInPhase)
    displayName: 'Run Playwright tests'
    env:
      BASE_URL: $(BASE_URL)
      TEST_USER_EMAIL: $(TEST_USER_EMAIL)
      TEST_USER_PASSWORD: $(TEST_USER_PASSWORD)
      PLAYWRIGHT_BROWSERS_PATH: $(Pipeline.Workspace)/.playwright-browsers

  - task: PublishTestResults@2
    condition: always()
    inputs:
      testResultsFormat: 'JUnit'
      testResultsFiles: 'results.xml'
      failTaskOnFailedTests: true

  - task: PublishBuildArtifacts@1
    condition: always()
    inputs:
      PathtoPublish: 'playwright-report'
      ArtifactName: 'playwright-report-$(System.JobPositionInPhase)'
```

### MCP (BDD) Mode

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pr:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  NODE_VERSION: '20'
  npm_config_cache: $(Pipeline.Workspace)/.npm

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: $(NODE_VERSION)
    displayName: 'Install Node.js'

  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      restoreKeys: |
        npm | "$(Agent.OS)"
      path: $(npm_config_cache)
    displayName: 'Cache npm packages'

  - script: npm ci --cache $(npm_config_cache) --prefer-offline
    displayName: 'Install dependencies'

  - script: npx playwright install --with-deps chromium
    displayName: 'Install Playwright browsers'

  - script: npx cucumber-js
    displayName: 'Run BDD tests'
    env:
      BASE_URL: $(BASE_URL)
      TEST_USER_EMAIL: $(TEST_USER_EMAIL)
      TEST_USER_PASSWORD: $(TEST_USER_PASSWORD)

  - task: PublishBuildArtifacts@1
    condition: always()
    inputs:
      PathtoPublish: 'reports'
      ArtifactName: 'cucumber-report'
```

**Setup:** Add `BASE_URL`, `TEST_USER_EMAIL`, and `TEST_USER_PASSWORD` under **Pipelines → Library → Variable Groups**, then link the variable group to this pipeline. Mark sensitive values as secret.

---

## CircleCI

Output file: `.circleci/config.yml`
Create the `.circleci/` directory if it does not exist.

### CLI Mode

```yaml
version: 2.1

orbs:
  node: circleci/node@5.2.0

executors:
  playwright:
    docker:
      - image: mcr.microsoft.com/playwright:v1.50.0-noble
    resource_class: medium

jobs:
  test:
    executor: playwright
    parallelism: 4
    steps:
      - checkout

      - node/install-packages:
          pkg-manager: npm
          cache-path: ~/.npm
          cache-version: v1

      - run:
          name: Install Playwright browsers
          command: npx playwright install chromium

      - run:
          name: Run Playwright tests (sharded)
          command: |
            SHARD="$((${CIRCLE_NODE_INDEX} + 1))"
            npx playwright test --shard=${SHARD}/${CIRCLE_NODE_TOTAL}
          environment:
            BASE_URL: $BASE_URL
            TEST_USER_EMAIL: $TEST_USER_EMAIL
            TEST_USER_PASSWORD: $TEST_USER_PASSWORD

      - store_artifacts:
          path: playwright-report
          destination: playwright-report-$CIRCLE_NODE_INDEX

      - store_test_results:
          path: results.xml

workflows:
  playwright-tests:
    jobs:
      - test:
          filters:
            branches:
              only:
                - main
                - develop
```

### MCP (BDD) Mode

```yaml
version: 2.1

orbs:
  node: circleci/node@5.2.0

executors:
  playwright:
    docker:
      - image: mcr.microsoft.com/playwright:v1.50.0-noble
    resource_class: medium

jobs:
  test:
    executor: playwright
    steps:
      - checkout

      - node/install-packages:
          pkg-manager: npm
          cache-path: ~/.npm
          cache-version: v1

      - run:
          name: Install Playwright browsers
          command: npx playwright install chromium

      - run:
          name: Run Cucumber BDD tests
          command: npx cucumber-js
          environment:
            BASE_URL: $BASE_URL
            TEST_USER_EMAIL: $TEST_USER_EMAIL
            TEST_USER_PASSWORD: $TEST_USER_PASSWORD

      - store_artifacts:
          path: reports
          destination: cucumber-report

workflows:
  bdd-tests:
    jobs:
      - test:
          filters:
            branches:
              only:
                - main
                - develop
```

**Setup:** Add `BASE_URL`, `TEST_USER_EMAIL`, and `TEST_USER_PASSWORD` as environment variables under **Project Settings → Environment Variables** in CircleCI.

---

## Bitbucket Pipelines

Output file: `bitbucket-pipelines.yml`

### CLI Mode

```yaml
image: mcr.microsoft.com/playwright:v1.50.0-noble

definitions:
  caches:
    npm: ~/.npm
    playwright: ~/.cache/ms-playwright

  steps:
    - step: &install
        name: Install dependencies
        caches:
          - npm
        script:
          - npm ci --cache ~/.npm --prefer-offline

    - step: &test-shard-1
        name: Test Shard 1/4
        caches:
          - playwright
        script:
          - npx playwright install chromium
          - npx playwright test --shard=1/4
        artifacts:
          - blob-report/**

    - step: &test-shard-2
        name: Test Shard 2/4
        caches:
          - playwright
        script:
          - npx playwright install chromium
          - npx playwright test --shard=2/4
        artifacts:
          - blob-report/**

    - step: &test-shard-3
        name: Test Shard 3/4
        caches:
          - playwright
        script:
          - npx playwright install chromium
          - npx playwright test --shard=3/4
        artifacts:
          - blob-report/**

    - step: &test-shard-4
        name: Test Shard 4/4
        caches:
          - playwright
        script:
          - npx playwright install chromium
          - npx playwright test --shard=4/4
        artifacts:
          - blob-report/**

    - step: &merge-reports
        name: Merge Reports
        script:
          - npm ci --cache ~/.npm --prefer-offline
          - npx playwright merge-reports --reporter html ./blob-report
        artifacts:
          - playwright-report/**

pipelines:
  default:
    - step: *install
    - parallel:
        - step: *test-shard-1
        - step: *test-shard-2
        - step: *test-shard-3
        - step: *test-shard-4
    - step: *merge-reports

  branches:
    main:
      - step: *install
      - parallel:
          - step: *test-shard-1
          - step: *test-shard-2
          - step: *test-shard-3
          - step: *test-shard-4
      - step: *merge-reports
```

### MCP (BDD) Mode

```yaml
image: mcr.microsoft.com/playwright:v1.50.0-noble

definitions:
  caches:
    npm: ~/.npm
    playwright: ~/.cache/ms-playwright

pipelines:
  default:
    - step:
        name: Run BDD Tests
        caches:
          - npm
          - playwright
        script:
          - npm ci --cache ~/.npm --prefer-offline
          - npx playwright install chromium
          - npx cucumber-js
        artifacts:
          - reports/**

  branches:
    main:
      - step:
          name: Run BDD Tests (main)
          caches:
            - npm
            - playwright
          script:
            - npm ci --cache ~/.npm --prefer-offline
            - npx playwright install chromium
            - npx cucumber-js
          artifacts:
            - reports/**
```

**Setup:** Add `BASE_URL`, `TEST_USER_EMAIL`, and `TEST_USER_PASSWORD` as repository variables under **Repository Settings → Pipelines → Repository variables** in Bitbucket.
