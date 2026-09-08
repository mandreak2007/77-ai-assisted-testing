# Jenkins — Pipeline Templates

Use these templates when the user selects Jenkins (option 3).
Output file: `Jenkinsfile` (at the project root)

Always replace every `<placeholder>` with a real value. Never leave angle-bracket placeholders in the final file.

---

## CLI Mode Template

Use when the project runner is `npx playwright test`.

```groovy
pipeline {
    agent {
        docker {
            image 'mcr.microsoft.com/playwright:v1.50.0-noble'
            args '--ipc=host'
        }
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    environment {
        BASE_URL          = credentials('PLAYWRIGHT_BASE_URL')
        TEST_USER_EMAIL   = credentials('PLAYWRIGHT_TEST_EMAIL')
        TEST_USER_PASSWORD = credentials('PLAYWRIGHT_TEST_PASSWORD')
        npm_config_cache  = "${WORKSPACE}/.npm"
    }

    stages {
        stage('Install dependencies') {
            steps {
                sh 'npm ci --cache .npm --prefer-offline'
            }
        }

        stage('Install Playwright browsers') {
            steps {
                sh 'npx playwright install chromium'
            }
        }

        stage('Run tests') {
            steps {
                sh 'npx playwright test'
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing         : true,
                        alwaysLinkToLastBuild: true,
                        keepAll              : true,
                        reportDir            : 'playwright-report',
                        reportFiles          : 'index.html',
                        reportName           : 'Playwright HTML Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'playwright-report/**/*', allowEmptyArchive: true
            cleanWs()
        }
        failure {
            echo 'Tests failed. Check the Playwright HTML Report for details.'
        }
    }
}
```

---

## MCP (BDD) Mode Template

Use when the project runner is `npx cucumber-js`.

```groovy
pipeline {
    agent {
        docker {
            image 'mcr.microsoft.com/playwright:v1.50.0-noble'
            args '--ipc=host'
        }
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    environment {
        BASE_URL           = credentials('PLAYWRIGHT_BASE_URL')
        TEST_USER_EMAIL    = credentials('PLAYWRIGHT_TEST_EMAIL')
        TEST_USER_PASSWORD = credentials('PLAYWRIGHT_TEST_PASSWORD')
        npm_config_cache   = "${WORKSPACE}/.npm"
    }

    stages {
        stage('Install dependencies') {
            steps {
                sh 'npm ci --cache .npm --prefer-offline'
            }
        }

        stage('Install Playwright browsers') {
            steps {
                sh 'npx playwright install chromium'
            }
        }

        stage('Run BDD tests') {
            steps {
                sh 'npx cucumber-js'
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing         : true,
                        alwaysLinkToLastBuild: true,
                        keepAll              : true,
                        reportDir            : 'reports',
                        reportFiles          : 'index.html',
                        reportName           : 'Cucumber HTML Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
            cleanWs()
        }
        failure {
            echo 'BDD tests failed. Check the Cucumber HTML Report for details.'
        }
    }
}
```

---

## Prerequisites

Before triggering this pipeline, set up the following Jenkins credentials:

| Credential ID | Type | Value |
|---|---|---|
| `PLAYWRIGHT_BASE_URL` | Secret text | Application base URL |
| `PLAYWRIGHT_TEST_EMAIL` | Secret text | Test account email |
| `PLAYWRIGHT_TEST_PASSWORD` | Secret text | Test account password |

Add them under **Manage Jenkins → Credentials → Global**.

## Notes

- The `--ipc=host` Docker flag is required for Chromium to run correctly in Jenkins.
- `publishHTML` requires the **HTML Publisher Plugin** — install it from the Jenkins Plugin Manager.
- `buildDiscarder` keeps only the last 20 builds to conserve disk space.
- `disableConcurrentBuilds` prevents multiple simultaneous runs on the same branch.
- To add parallel sharding in Jenkins, replace the single `Run tests` stage with a `parallel` block containing multiple `sh 'npx playwright test --shard=N/4'` stages and a downstream merge job.
- Update the Playwright image tag to match your `@playwright/test` version.
