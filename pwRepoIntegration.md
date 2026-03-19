# Playwright E2E Testing + GitHub Actions CI Guide

A practical blueprint for adding **Playwright E2E tests** to a repo and wiring them into **GitHub Actions CI**, including test management integration (TestRail), custom reporting, and advanced CI patterns.

This guide covers:

1. What you're adding
2. Recommended project structure
3. Install Playwright
4. A good Playwright config
5. Folder conventions
6. How to write tests well
7. Page objects
8. Fixtures
9. Authentication strategy
10. Test data and environment strategy
11. Reports, traces, screenshots, and artifacts
12. GitHub Actions CI setup
13. Build step considerations
14. CI evolution path
15. Reports in GitHub Actions
16. Practical folder strategy by repo size
17. Page objects vs helpers vs fixtures
18. Common mistakes
19. A ready-to-copy starter
20. Opinionated recommendations
21. Suggested implementation order
22. TestRail integration
23. Custom HTML report from the repo
24. Publishing reports via GitHub Pages
25. `.gitignore` for Playwright projects
26. Advanced CI patterns

---

## 1) What you’re adding

A solid Playwright E2E setup usually includes:

- **Playwright dependency**
- **Playwright config**
- **E2E test folder**
- Optional:
  - **page objects**
  - **fixtures**
  - **helpers**
  - **test data factories**
- **Local HTML report**
- **CI reporters** for GitHub Actions / JUnit
- A **GitHub Actions workflow**
- A strategy for:
  - starting the app
  - waiting for readiness
  - seeding data
  - cleaning state
  - handling secrets/env vars

---

## 2) Recommended project structure

A clean structure for most repos:

```text
repo/
├─ src/
├─ tests/
│  ├─ e2e/
│  │  ├─ auth.spec.ts
│  │  ├─ checkout.spec.ts
│  │  ├─ smoke/
│  │  │  └─ homepage.spec.ts
│  │  ├─ fixtures/
│  │  │  └─ test-fixtures.ts
│  │  ├─ pages/
│  │  │  ├─ login-page.ts
│  │  │  └─ dashboard-page.ts
│  │  ├─ utils/
│  │  │  ├─ api-helpers.ts
│  │  │  └─ test-data.ts
│  │  └─ global/
│  │     ├─ setup.ts
│  │     └─ teardown.ts
├─ playwright.config.ts
├─ package.json
└─ .github/
   └─ workflows/
      └─ e2e.yml
```

### Why this works

- `tests/e2e/*.spec.ts`: actual scenarios
- `tests/e2e/pages/`: page objects for reusable UI interactions
- `tests/e2e/fixtures/`: shared fixtures for authenticated users, seeded state, etc.
- `tests/e2e/utils/`: helpers, API calls, data builders
- `tests/e2e/global/`: setup/teardown logic
- `playwright.config.ts`: all runner config in one place
- `.github/workflows/e2e.yml`: CI automation

If the repo is large, you can go further:

```text
tests/
└─ e2e/
   ├─ smoke/
   ├─ regression/
   ├─ mobile/
   ├─ accessibility/
   └─ ...
```

That helps when splitting suites in CI.

---

## 3) Install Playwright

For a Node.js repo:

```bash
npm install -D @playwright/test
npx playwright install
```

If you want the browser system dependencies too for CI/Linux images:

```bash
npx playwright install --with-deps
```

Add scripts to `package.json`:

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:headed": "playwright test --headed",
    "test:e2e:debug": "playwright test --debug",
    "test:e2e:report": "playwright show-report"
  }
}
```

---

## 4) A good Playwright config

Here’s a strong baseline config:

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  timeout: 30_000,
  expect: {
    timeout: 5_000,
  },

  reporter: process.env.CI
    ? [
        ['list'],
        ['html', { outputFolder: 'playwright-report', open: 'never' }],
        ['junit', { outputFile: 'test-results/junit.xml' }],
        ['github']
      ]
    : [
        ['list'],
        ['html', { outputFolder: 'playwright-report', open: 'on-failure' }]
      ],

  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL || 'http://127.0.0.1:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 10_000,
    navigationTimeout: 15_000,
  },

  webServer: {
    command: 'npm run start:test',
    url: 'http://127.0.0.1:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    }
  ],

  outputDir: 'test-results/artifacts',
});
```

### Why these settings matter

- `forbidOnly`: prevents accidental `.only` in CI
- `retries`: helps reduce transient flake
- `workers`: limits load in CI
- `trace: 'on-first-retry'`: best cost/benefit debugging choice
- `screenshot/video`: keep artifacts only when useful
- `webServer`: starts your app automatically before tests
- `baseURL`: makes test navigation simpler

---

## 5) What folder conventions I recommend

### Minimal setup

If your app is small, start with:

```text
tests/e2e/
  homepage.spec.ts
  login.spec.ts
playwright.config.ts
```

### Scalable setup

For medium/large apps:

```text
tests/e2e/
  smoke/
  auth/
  billing/
  admin/
  pages/
  fixtures/
  utils/
```

### Naming guidance

- Test files: `feature-name.spec.ts`
- Page objects: `feature-page.ts`
- Fixtures: `test-fixtures.ts`
- Helpers: `something-helper.ts` or `api.ts`

---

## 6) How to write tests well

A simple E2E test:

```typescript
import { test, expect } from '@playwright/test';

test('homepage loads', async ({ page }) => {
  await page.goto('/');
  await expect(page.getByRole('heading', { name: /welcome/i })).toBeVisible();
});
```

### Best practices

- Use **accessible locators** first:
  - `getByRole`
  - `getByLabel`
  - `getByText` sparingly
  - `getByTestId` when needed
- Avoid brittle CSS selectors
- Keep each test independent
- Assert user-visible outcomes, not implementation details

Example with interactions:

```typescript
import { test, expect } from '@playwright/test';

test('user can sign in', async ({ page }) => {
  await page.goto('/login');

  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('supersecret');
  await page.getByRole('button', { name: /sign in/i }).click();

  await expect(page).toHaveURL(/dashboard/);
  await expect(page.getByRole('heading', { name: /dashboard/i })).toBeVisible();
});
```

---

## 7) Page objects: when to use them

Use page objects when:

- a page has repeated interactions
- flows are reused in multiple tests
- selectors are complex

Example:

```typescript
import { expect, Page } from '@playwright/test';

export class LoginPage {
  constructor(private readonly page: Page) {}

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: /sign in/i }).click();
  }

  async expectLoaded() {
    await expect(this.page.getByRole('heading', { name: /sign in/i })).toBeVisible();
  }
}
```

Use it in a test:

```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login-page';

test('user can sign in', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.expectLoaded();
  await loginPage.login('user@example.com', 'supersecret');

  await expect(page).toHaveURL(/dashboard/);
});
```

### Don’t overdo page objects

Bad pattern:
- putting assertions, business logic, and test intent all inside page classes

Good pattern:
- page object handles UI interactions
- test keeps scenario meaning clear

---

## 8) Fixtures: how to avoid duplicated setup

Fixtures are ideal for authenticated states, pre-created users, seeded data, and reusable context.

Example custom fixture:

```typescript
import { test as base } from '@playwright/test';

type TestFixtures = {
  testUser: {
    email: string;
    password: string;
  };
};

export const test = base.extend<TestFixtures>({
  testUser: async ({}, use) => {
    await use({
      email: 'user@example.com',
      password: 'supersecret',
    });
  },
});

export { expect } from '@playwright/test';
```

Use it:

```typescript
import { test, expect } from '../fixtures/test-fixtures';
import { LoginPage } from '../pages/login-page';

test('user can sign in', async ({ page, testUser }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login(testUser.email, testUser.password);

  await expect(page).toHaveURL(/dashboard/);
});
```

---

## 9) Authentication strategy

For E2E, login is often the slowest repeated step. You generally have 3 options:

### Option A: Login through UI in every test

Good for:
- login coverage itself
- very small suites

Bad for:
- speed
- flakiness

### Option B: Save authenticated storage state

Best for many repos.

Example idea:
- one setup script logs in once
- save `storageState`
- reuse in tests

Config:

```typescript
use: {
  storageState: 'tests/e2e/.auth/user.json',
}
```

Setup test:

```typescript
import { chromium } from '@playwright/test';

async function globalSetup() {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto('http://127.0.0.1:3000/login');
  await page.getByLabel('Email').fill(process.env.E2E_EMAIL!);
  await page.getByLabel('Password').fill(process.env.E2E_PASSWORD!);
  await page.getByRole('button', { name: /sign in/i }).click();

  await page.context().storageState({ path: 'tests/e2e/.auth/user.json' });
  await browser.close();
}

export default globalSetup;
```

Then in config:

```typescript
globalSetup: './tests/e2e/global/setup.ts',
```

### Option C: Login via API

Often the best if your backend supports it.
- faster
- less flaky
- ideal for non-auth-specific tests

---

## 10) Test data and environment strategy

You need stable data.

### Best options

- dedicated **test environment**
- deterministic seeded DB
- disposable user accounts / fixtures
- API helpers to create data on demand

Example utility:

```typescript
export async function createOrderViaApi(request: any, token: string) {
  const response = await request.post('/api/orders', {
    headers: { Authorization: `Bearer ${token}` },
    data: {
      item: 'Test item',
      quantity: 1
    }
  });

  return response.json();
}
```

### Avoid

- relying on shared mutable staging data
- depending on random preexisting records
- tests that fail if run in parallel

---

## 11) Reports, traces, screenshots, and artifacts

A good Playwright setup should produce:

- **HTML report** for humans
- **JUnit XML** for CI integrations
- **trace.zip** for debugging failures
- screenshots/videos only on failure
- **Test management integration** (e.g., TestRail) for traceability

Recommended outputs:

```text
playwright-report/      # built-in HTML report
test-results/           # JUnit XML, traces, screenshots, videos
custom-report/          # optional: your own HTML dashboard
```

Suggested policy:

- Local:
  - HTML report
  - open on failure
- CI:
  - HTML report uploaded as artifact
  - JUnit XML for test result integration
  - traces/screenshots/videos uploaded on failure
  - push results to test management tools (TestRail, Xray, etc.)

### Understanding Playwright reporters

Playwright supports multiple reporters running simultaneously. Key types:

| Reporter | Purpose | Audience |
|----------|---------|----------|
| `list` | Console output during run | Developer |
| `html` | Interactive browsable report | QA / Developer |
| `junit` | Machine-readable XML | CI systems |
| `json` | Full JSON output | Custom tooling |
| `github` | GitHub Actions annotations | PR reviewers |
| `blob` | Mergeable binary for sharded runs | CI pipelines |
| Custom | Any format you need | Stakeholders |

### Trace viewer: the most powerful debugging tool

The Playwright trace viewer is often underused. A trace captures:
- DOM snapshots at every action
- network requests
- console logs
- action timeline with before/after screenshots

To view a trace locally:

```bash
npx playwright show-trace test-results/artifacts/test-name/trace.zip
```

Or drag and drop `trace.zip` into https://trace.playwright.dev.

---

## 12) GitHub Actions CI setup

A simple and strong workflow:

```yaml
name: E2E Tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run E2E tests
        run: npm run test:e2e
        env:
          CI: true
          PLAYWRIGHT_BASE_URL: http://127.0.0.1:3000
          E2E_EMAIL: ${{ secrets.E2E_EMAIL }}
          E2E_PASSWORD: ${{ secrets.E2E_PASSWORD }}

      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report
          retention-days: 14

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results
          retention-days: 14
```

### Important notes

- `if: always()` ensures artifacts upload even if tests fail
- use GitHub Secrets for credentials
- keep CI timeout explicit
- run on `pull_request` at minimum

### Caching Playwright browsers in CI

Downloading browsers on every run can add 1-2 minutes. Cache them:

```yaml
      - name: Get Playwright version
        id: pw-version
        run: echo "version=$(npx playwright --version | awk '{print $2}')" >> $GITHUB_OUTPUT

      - name: Cache Playwright browsers
        uses: actions/cache@v4
        id: playwright-cache
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ steps.pw-version.outputs.version }}

      - name: Install Playwright browsers
        if: steps.playwright-cache.outputs.cache-hit != 'true'
        run: npx playwright install --with-deps

      - name: Install Playwright system deps only
        if: steps.playwright-cache.outputs.cache-hit == 'true'
        run: npx playwright install-deps
```

This way browsers are only downloaded when the Playwright version changes.

### Adding PR comments with test results

Use a GitHub Actions step to post test summary directly on PRs:

```yaml
      - name: Test results summary
        if: always()
        uses: daun/playwright-report-summary@v3
        with:
          report-file: test-results/junit.xml
```

This posts a comment on the PR with pass/fail counts and failure details — much faster than downloading artifacts.

---

## 13) If your app needs a build step

Many apps need:

```yaml
- name: Build app
  run: npm run build
```

And maybe a dedicated start command:

```json
{
  "scripts": {
    "start:test": "npm run build && npm run start"
  }
}
```

But often better is:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "start:test": "next start -p 3000"
  }
}
```

For CI, prefer a predictable production-like server over hot-reload dev mode unless you truly need dev mode.

---

## 14) Recommended CI evolution path

### Stage 1: One browser, smoke tests only

Start with:
- Chromium only
- a few critical flows:
  - homepage
  - login
  - checkout/core action

### Stage 2: Add regression suite

- broader flows
- more fixtures
- stable seeded data

### Stage 3: Parallelization and matrix

Split by:
- browser
- suite
- app area

Example future matrix:

```yaml
strategy:
  matrix:
    project: [chromium, firefox]
```

Then run:

```yaml
run: npx playwright test --project=${{ matrix.project }}
```

Only do this after the suite is stable.

### Stage 4: Scheduled runs and notifications

Run E2E on a schedule to catch environment or deployment issues:

```yaml
on:
  schedule:
    - cron: '0 6 * * 1-5'  # weekdays at 06:00 UTC
  workflow_dispatch:        # allow manual triggers
```

Add Slack or Teams notification on failure:

```yaml
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "E2E tests failed on ${{ github.ref_name }} — ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Stage 5: Docker-based CI for consistency

For full reproducibility, run tests inside a Docker container:

```yaml
    container:
      image: mcr.microsoft.com/playwright:v1.49.0-noble
```

This avoids dependency drift between CI runners and guarantees all system deps are present.

---

## 15) Reports in GitHub Actions: what to keep

I recommend:

- **artifact upload always**
- **JUnit XML** if your org consumes machine-readable test output
- **HTML report** for debugging
- **traces on retry/failure**

If you want PR visibility, you can later add:
- job summaries
- annotations
- a test reporter integration

But the basic artifact model is enough to start.

---

## 16) Practical folder strategy by repo size

### Small repo

```text
tests/e2e/
  login.spec.ts
  homepage.spec.ts
playwright.config.ts
```

### Medium repo

```text
tests/e2e/
  smoke/
  auth/
  checkout/
  pages/
  fixtures/
  utils/
```

### Large repo / monorepo

```text
apps/web/
  src/
  tests/e2e/
  playwright.config.ts
.github/workflows/e2e-web.yml
```

In a monorepo, keep E2E tests close to the app they validate.

---

## 17) What to put in page objects vs helpers vs fixtures

### Page objects

Use for:
- selectors
- page interactions
- reusable page-level actions

### Helpers / utils

Use for:
- random data generation
- API setup
- date formatting
- polling utilities

### Fixtures

Use for:
- authenticated user context
- seeded entities
- shared setup objects

### Tests

Use for:
- scenario narrative
- assertions
- orchestration

A good rule:
- if reused across tests, extract it
- if only used once and simple, keep it in the test

---

## 18) Common mistakes

### 1. Using brittle selectors

Bad:

```ts
page.locator('.btn.primary:nth-child(2)')
```

Better:

```ts
page.getByRole('button', { name: /submit/i })
```

### 2. Adding arbitrary sleeps

Bad:

```ts
await page.waitForTimeout(5000);
```

Better:
- wait for visible UI
- wait for URL
- wait for API response if necessary

### 3. Shared state between tests

Each test should be runnable alone.

### 4. Testing too much in one test

Prefer shorter focused scenarios.

### 5. Running full cross-browser too early

Get Chromium green first.

### 6. Using staging as the only data source

Create stable, controllable test data.

### 7. No artifact collection

Without traces/screenshots/report, debugging CI failures is painful.

---

## 19) A ready-to-copy starter

### `package.json` scripts

```json
{
  "scripts": {
    "start:test": "npm run start",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:report": "playwright show-report"
  }
}
```

### `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  forbidOnly: !!process.env.CI,
  reporter: [
    ['list'],
    ['html', { outputFolder: 'playwright-report', open: 'never' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['github'],
  ],
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL || 'http://127.0.0.1:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  webServer: {
    command: 'npm run start:test',
    url: 'http://127.0.0.1:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
  outputDir: 'test-results/artifacts',
});
```

### Example test

```typescript
import { test, expect } from '@playwright/test';

test('homepage renders', async ({ page }) => {
  await page.goto('/');
  await expect(page.getByRole('main')).toBeVisible();
});
```

### CI workflow

```yaml
name: E2E Tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run test:e2e
        env:
          CI: true
          PLAYWRIGHT_BASE_URL: http://127.0.0.1:3000

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results
```

---

## 20) My opinionated recommendations

If you want the simplest setup that still scales well:

- Put tests in `tests/e2e`
- Group by feature: `auth`, `billing`, `smoke`
- Use **Chromium only first**
- Use **page objects lightly**
- Use **fixtures for auth and data**
- Use **HTML + JUnit + GitHub reporter**
- Upload reports/artifacts in CI with `if: always()`
- Use a dedicated `start:test` script
- Prefer **stable test env + seeded data**
- Enable `trace: on-first-retry`

---

## 21) Suggested implementation order

1. Install Playwright
2. Add `playwright.config.ts`
3. Add `tests/e2e/smoke/homepage.spec.ts`
4. Add `start:test` script
5. Make local run green
6. Add GitHub Actions workflow
7. Upload artifacts
8. Add auth fixture or storage state
9. Add more feature tests
10. Later add matrix/cross-browser

---

## 22) TestRail integration

TestRail is a popular test management tool. Integrating Playwright results into TestRail gives:

- **traceability** between automated tests and test cases
- **historical pass/fail trends**
- **visibility for non-technical stakeholders**
- **release readiness dashboards**

### How it works

1. Each Playwright test maps to a TestRail test case by ID
2. After a test run, results are pushed to TestRail via API
3. TestRail shows pass/fail per case, with links back to CI

### Option A: Use a TestRail reporter package

Install:

```bash
npm install -D playwright-testrail-reporter
```

Add to `playwright.config.ts`:

```typescript
reporter: [
  ['list'],
  ['html', { outputFolder: 'playwright-report', open: 'never' }],
  ['playwright-testrail-reporter', {
    host: 'https://yourcompany.testrail.io',
    username: process.env.TESTRAIL_USERNAME,
    password: process.env.TESTRAIL_API_KEY,
    projectId: 1,
    suiteId: 1,
    runName: `E2E Run - ${new Date().toISOString()}`,
  }],
],
```

Then tag your tests with TestRail case IDs:

```typescript
test('C1234 user can sign in', async ({ page }) => {
  // The "C1234" prefix maps to TestRail case ID 1234
  await page.goto('/login');
  // ...
});
```

### Option B: Push results via TestRail API directly

If you prefer more control, use TestRail's REST API after tests complete:

```typescript
// tests/e2e/utils/testrail-client.ts
export class TestRailClient {
  private baseUrl: string;
  private headers: Record<string, string>;

  constructor(host: string, username: string, apiKey: string) {
    this.baseUrl = `${host}/index.php?/api/v2`;
    this.headers = {
      'Content-Type': 'application/json',
      'Authorization': `Basic ${Buffer.from(`${username}:${apiKey}`).toString('base64')}`,
    };
  }

  async addRun(projectId: number, suiteId: number, name: string) {
    const res = await fetch(`${this.baseUrl}/add_run/${projectId}`, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify({ suite_id: suiteId, name, include_all: false, case_ids: [] }),
    });
    return res.json();
  }

  async addResultForCase(runId: number, caseId: number, statusId: number, comment: string) {
    // statusId: 1 = Passed, 5 = Failed
    const res = await fetch(`${this.baseUrl}/add_result_for_case/${runId}/${caseId}`, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify({ status_id: statusId, comment }),
    });
    return res.json();
  }
}
```

Use it in a global teardown:

```typescript
// tests/e2e/global/teardown.ts
import { TestRailClient } from '../utils/testrail-client';

async function globalTeardown() {
  if (!process.env.TESTRAIL_PUSH) return;

  const client = new TestRailClient(
    process.env.TESTRAIL_HOST!,
    process.env.TESTRAIL_USERNAME!,
    process.env.TESTRAIL_API_KEY!,
  );

  // Parse JUnit XML or JSON results and push to TestRail
  // ...
}

export default globalTeardown;
```

### CI environment variables for TestRail

```yaml
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          CI: true
          TESTRAIL_PUSH: true
          TESTRAIL_HOST: https://yourcompany.testrail.io
          TESTRAIL_USERNAME: ${{ secrets.TESTRAIL_USERNAME }}
          TESTRAIL_API_KEY: ${{ secrets.TESTRAIL_API_KEY }}
```

### Best practices for TestRail integration

- Keep case IDs in test titles (`C1234`) or as annotations
- Only push to TestRail in CI, not locally
- Create a new TestRail run per CI pipeline execution
- Link the CI run URL in the TestRail run description
- Use TestRail milestones to group runs by release

---

## 23) Custom HTML report from the repo

The built-in Playwright HTML report is great for developers, but stakeholders and QA leads sometimes want a **custom dashboard** with:

- a summary with pass/fail/skip counts
- execution time per suite
- links to traces and screenshots
- environment info (branch, commit, timestamp)
- a format that matches the team's branding

### How to build a custom report

Playwright can output results as JSON. Use that to generate any report format.

#### Step 1: Add the JSON reporter

```typescript
reporter: [
  ['list'],
  ['json', { outputFile: 'test-results/results.json' }],
  ['html', { outputFolder: 'playwright-report', open: 'never' }],
],
```

#### Step 2: Create a report generator script

```typescript
// scripts/generate-report.ts
import * as fs from 'fs';
import * as path from 'path';

interface TestResult {
  title: string;
  status: string;
  duration: number;
  error?: { message: string };
}

interface Suite {
  title: string;
  specs: Array<{
    title: string;
    tests: Array<{
      results: TestResult[];
    }>;
  }>;
  suites?: Suite[];
}

function collectResults(suite: Suite, results: TestResult[] = []): TestResult[] {
  for (const spec of suite.specs || []) {
    for (const test of spec.tests || []) {
      for (const result of test.results || []) {
        results.push({
          title: spec.title,
          status: result.status,
          duration: result.duration,
          error: result.error,
        });
      }
    }
  }
  for (const child of suite.suites || []) {
    collectResults(child, results);
  }
  return results;
}

function generateHtml(results: TestResult[]): string {
  const passed = results.filter(r => r.status === 'passed').length;
  const failed = results.filter(r => r.status === 'failed').length;
  const skipped = results.filter(r => r.status === 'skipped').length;
  const totalDuration = results.reduce((sum, r) => sum + r.duration, 0);

  const rows = results.map(r => `
    <tr class="${r.status}">
      <td>${r.title}</td>
      <td><span class="badge ${r.status}">${r.status}</span></td>
      <td>${(r.duration / 1000).toFixed(2)}s</td>
      <td>${r.error?.message || '—'}</td>
    </tr>
  `).join('');

  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>E2E Test Report</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 2rem; background: #f8f9fa; }
    h1 { color: #1a1a2e; }
    .summary { display: flex; gap: 1rem; margin: 1rem 0 2rem; }
    .summary .card {
      padding: 1rem 1.5rem; border-radius: 8px; color: white; font-size: 1.1rem;
    }
    .card.passed { background: #22c55e; }
    .card.failed { background: #ef4444; }
    .card.skipped { background: #f59e0b; }
    .card.time { background: #3b82f6; }
    table { width: 100%; border-collapse: collapse; background: white; border-radius: 8px; overflow: hidden; }
    th, td { padding: 0.75rem 1rem; text-align: left; border-bottom: 1px solid #e5e7eb; }
    th { background: #1a1a2e; color: white; }
    .badge { padding: 0.25rem 0.75rem; border-radius: 4px; color: white; font-size: 0.85rem; }
    .badge.passed { background: #22c55e; }
    .badge.failed { background: #ef4444; }
    .badge.skipped { background: #f59e0b; }
    tr.failed { background: #fef2f2; }
    .meta { color: #6b7280; font-size: 0.9rem; margin-bottom: 1rem; }
  </style>
</head>
<body>
  <h1>E2E Test Report</h1>
  <p class="meta">
    Generated: ${new Date().toISOString()} |
    Branch: ${process.env.GITHUB_REF_NAME || 'local'} |
    Commit: ${process.env.GITHUB_SHA?.slice(0, 7) || 'N/A'}
  </p>
  <div class="summary">
    <div class="card passed">Passed: ${passed}</div>
    <div class="card failed">Failed: ${failed}</div>
    <div class="card skipped">Skipped: ${skipped}</div>
    <div class="card time">Duration: ${(totalDuration / 1000).toFixed(1)}s</div>
  </div>
  <table>
    <thead><tr><th>Test</th><th>Status</th><th>Duration</th><th>Error</th></tr></thead>
    <tbody>${rows}</tbody>
  </table>
</body>
</html>`;
}

const raw = fs.readFileSync('test-results/results.json', 'utf-8');
const report = JSON.parse(raw);
const results = collectResults(report.suites?.[0] || report);
const html = generateHtml(results);

const outDir = 'custom-report';
fs.mkdirSync(outDir, { recursive: true });
fs.writeFileSync(path.join(outDir, 'index.html'), html);
console.log(`Custom report generated at ${outDir}/index.html`);
```

#### Step 3: Add a script to `package.json`

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:report": "playwright show-report",
    "test:e2e:custom-report": "npx tsx scripts/generate-report.ts"
  }
}
```

#### Step 4: Run it after tests in CI

```yaml
      - name: Generate custom report
        if: always()
        run: npm run test:e2e:custom-report

      - name: Upload custom report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: custom-report
          path: custom-report
```

This gives you full control over what the report shows and how it looks.

---

## 24) Publishing reports via GitHub Pages

Instead of downloading artifacts, you can publish the HTML report to GitHub Pages for easy browsing.

### Setup

1. Enable GitHub Pages in repo settings (source: GitHub Actions)
2. Add a deploy step to your workflow:

```yaml
  deploy-report:
    needs: e2e
    if: always()
    runs-on: ubuntu-latest

    permissions:
      pages: write
      id-token: write

    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Download report artifact
        uses: actions/download-artifact@v4
        with:
          name: playwright-report
          path: report

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v4
        with:
          path: report

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

After this, every CI run publishes the latest report to `https://<org>.github.io/<repo>/`.

### When to use this

- teams that want **instant report access** without downloading artifacts
- QA leads who review reports frequently
- projects with stakeholders who don't use GitHub daily

### Considerations

- GitHub Pages shows only the **latest** deployment (not historical)
- for history, push to a subfolder per run or use a dedicated reporting service
- if the repo is private, GitHub Pages will also be private (GitHub Enterprise or Pro required)

---

## 25) `.gitignore` for Playwright projects

Add these entries to your `.gitignore` to avoid committing test artifacts:

```gitignore
# Playwright
test-results/
playwright-report/
custom-report/
blob-report/
tests/e2e/.auth/
```

### Why each matters

- `test-results/`: traces, screenshots, videos, JUnit XML — all generated per run
- `playwright-report/`: built-in HTML report output
- `custom-report/`: your custom report output (if using one)
- `blob-report/`: used for sharded CI runs before merging
- `tests/e2e/.auth/`: stored authentication state contains session tokens — **never commit this**

---

## 26) Advanced CI patterns

### Sharding for large test suites

When your suite grows, split across multiple CI machines:

```yaml
    strategy:
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]

    steps:
      # ... setup steps ...

      - name: Run E2E tests (shard)
        run: npx playwright test --shard=${{ matrix.shard }}
```

Then merge reports:

```yaml
  merge-reports:
    needs: e2e
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Download all shard reports
        uses: actions/download-artifact@v4
        with:
          pattern: blob-report-*
          merge-multiple: true
          path: all-blob-reports

      - name: Merge reports
        run: npx playwright merge-reports --reporter=html ./all-blob-reports

      - name: Upload merged report
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report
```

### Branch protection with required E2E checks

Enforce that E2E tests pass before merging:

1. In GitHub repo settings → Branches → Branch protection rules
2. Enable **Require status checks to pass before merging**
3. Add the `e2e` job as a required check

This prevents regressions from reaching `main`.

### Environment-specific CI runs

Run E2E against different environments:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        description: Target environment
        options:
          - staging
          - production
        default: staging

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      # ...
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          PLAYWRIGHT_BASE_URL: ${{ inputs.environment == 'production' && 'https://app.example.com' || 'https://staging.example.com' }}
```

### Tagging tests for selective execution

Use Playwright's `grep` to run subsets:

```bash
# Run only smoke tests
npx playwright test --grep @smoke

# Skip slow tests
npx playwright test --grep-invert @slow
```

Tag your tests:

```typescript
test('@smoke homepage loads', async ({ page }) => {
  await page.goto('/');
  await expect(page.getByRole('main')).toBeVisible();
});

test('@regression user profile shows order history', async ({ page }) => {
  // ...
});
```

In CI, you can use this to run smoke tests on every PR and full regression on a schedule.