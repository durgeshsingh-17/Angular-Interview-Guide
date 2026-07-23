# Chapter 72: CI/CD

## Scenario 1: The 45-minute pipeline in a large Nx monorepo

**Situation:** Your organization runs a single Nx-based monorepo containing 40 Angular applications and 120 shared libraries. Every PR triggers a pipeline that lints, builds, and tests the *entire* workspace, regardless of what changed. A one-line CSS fix in a leaf library now takes 45 minutes to get green CI, and engineers have started merging on red checks "because it's probably unrelated."

**Question:** How would you redesign this pipeline so CI only builds/tests what the change actually affects, while still guaranteeing nothing downstream silently breaks?

**Answer:**
The core fix is to stop treating the monorepo as one unit and start using Nx's dependency graph to compute the *affected* set relative to a stable base, then run tasks only for that set, with distributed caching so repeated runs of the same input never recompute.

1. **Establish a correct base/head comparison.** In GitHub Actions this means fetching enough git history to diff against the target branch (`fetch-depth: 0` or Nx's `nrwl/nx-set-shas` action), not just `HEAD~1`.

```yaml
name: CI
on:
  pull_request:
    branches: [main]

jobs:
  affected:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - uses: nrwl/nx-set-shas@v4   # sets NX_BASE / NX_HEAD correctly

      - name: Lint affected
        run: npx nx affected -t lint --parallel=3

      - name: Test affected
        run: npx nx affected -t test --parallel=3 --configuration=ci

      - name: Build affected
        run: npx nx affected -t build --parallel=3
```

2. **Use Nx Cloud (or a self-hosted remote cache) for distributed task caching.** Two engineers changing unrelated libraries on the same day both trigger `nx affected -t build`; if lib-a was already built with identical inputs (source hash, compiler options, dependency versions) on any agent, the cache serves the result instantly instead of rebuilding.

```bash
npx nx affected -t build --parallel=3 --nxCloud=true
```

3. **Guarantee downstream safety with the project graph, not guesswork.** Nx's `affected` isn't "files that changed" — it's "projects whose input hash changed," computed by walking the dependency graph from changed files upward through every `implicitDependency` and `import`. If `shared-ui` changes, every app that imports it (transitively) is included automatically. This is the guarantee that lets you *not* run everything.

4. **Add a nightly/scheduled full-graph run** as a safety net for things `affected` can't see (e.g., a change to a global `tsconfig.base.json` path mapping that Nx *does* detect via `nx.json` `implicitDependencies`, but external factors like a registry version bump might not).

5. **Tradeoffs to discuss in interview:**
   - Affected-only CI trades a small risk of missing an edge case (e.g., a runtime-only dependency Nx can't statically see) for a 10x+ speedup — mitigate with the nightly full build.
   - Distributed caching requires trusting cache correctness; a bad cache key (e.g., not including environment variables that affect the build) causes "works with cache, breaks without" bugs — so cache keys (`namedInputs` in `nx.json`) must include all env vars and config files that influence output.
   - `--parallel=3` needs tuning against runner CPU count — Nx recommends `--parallel` roughly equal to available cores minus one.

**Interviewer intent:** Tests whether the candidate understands dependency-graph-based CI scoping (not just "run changed files' tests") and can reason about cache correctness and safety nets, not just speed.

---

## Scenario 2: Flaky E2E tests blocking every PR

**Situation:** Your team has a Playwright E2E suite of ~300 specs running against a deployed preview environment. Roughly 15 of those specs fail intermittently — sometimes due to real timing bugs, sometimes due to test environment flakiness (shared test data, network jitter). Because the pipeline requires 100% pass to merge, engineers are re-running the whole CI job 3–4 times per PR, wasting hours of compute and eroding trust in "red" results.

**Question:** How do you stabilize this so CI signal is trustworthy again, without simply disabling the flaky tests forever?

**Answer:**
The goal is to separate "flaky infrastructure" from "test suite health" as two distinct problems, and stop conflating a failing merge gate with an untriaged flake list.

1. **Quarantine, don't delete.** Tag flaky specs (`@flaky` tag or a separate `flaky.spec.ts` project) and move them out of the blocking pipeline into a non-blocking "flaky watch" job that still runs on every PR and reports results to a dashboard, but doesn't gate merge.

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'stable', testMatch: /.*\.spec\.ts/, testIgnore: /.*\.flaky\.spec\.ts/ },
    { name: 'flaky-watch', testMatch: /.*\.flaky\.spec\.ts/ },
  ],
});
```

```yaml
jobs:
  e2e-stable:
    runs-on: ubuntu-latest
    steps:
      - run: npx playwright test --project=stable
        # this job is a required status check

  e2e-flaky-watch:
    runs-on: ubuntu-latest
    continue-on-error: true   # never blocks merge
    steps:
      - run: npx playwright test --project=flaky-watch
      - run: npx playwright show-report  # upload as artifact for triage
```

2. **Enable built-in retry with visibility, not silence.** Playwright's `retries: 2` masks a single flaky rerun as green, but you must still surface "this test needed a retry" so it doesn't rot forever.

```ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
  reporter: [['html'], ['json', { outputFile: 'results.json' }]],
});
```
Post-process `results.json` in CI to post a Slack/GitHub comment: "3 specs passed only after retry — investigate: login.spec.ts, checkout.spec.ts."

3. **Fix root causes systematically, not test-by-test:**
   - **Shared test data races** → give each test worker an isolated tenant/user (Playwright's `test.describe.configure({ mode: 'parallel' })` plus per-worker fixtures that create/teardown their own data) instead of a shared seeded database.
   - **Network/animation timing** → replace hard `waitForTimeout` calls with `waitForSelector`/`expect(locator).toBeVisible()` auto-retrying assertions, and disable CSS animations in the test environment (`prefers-reduced-motion` + a test-only style override) so timing isn't tied to transition duration.
   - **Environment nondeterminism** → pin the preview environment's clock/timezone and seed random data generators with a fixed seed per test run.

4. **Track a flake rate SLO.** A test that fails >2% of runs over the last 50 runs auto-files a ticket (via a small script parsing historical CI results) and gets quarantined until fixed — this keeps the "flaky watch" bucket from becoming a graveyard.

5. **Shard for both speed and isolation.**

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx playwright test --project=stable --shard=${{ matrix.shard }}/4
```

**Interviewer intent:** Tests whether the candidate distinguishes "flaky test" from "flaky infrastructure," and whether they'd reach for a real engineering fix (isolation, deterministic waits) instead of just cranking up `retries`.

---

## Scenario 3: Enforcing bundle-size budgets in CI

**Situation:** A customer-facing Angular app has crept from 380KB to 640KB of initial JS over six months, with no single PR being the obvious culprit. Leadership wants a hard ceiling enforced in CI going forward, plus visibility into what's driving growth.

**Question:** How do you implement bundle budgets that actually block regressions, and how do you give developers actionable feedback instead of just a red X?

**Answer:**
Angular has two complementary layers here: the **build-time budget system** in `angular.json` (fails the build outright) and a **CI-level bundle diff/report** (gives PR-level visibility before merge).

1. **Configure budgets in `angular.json`** so the build itself fails past a threshold — this is the hard gate, enforced identically in local dev and CI (no drift):

```json
{
  "projects": {
    "app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kb",
                  "maximumError": "600kb"
                },
                {
                  "type": "anyComponentStyle",
                  "maximumWarning": "4kb",
                  "maximumError": "8kb"
                },
                {
                  "type": "bundle",
                  "name": "polyfills",
                  "maximumError": "50kb"
                }
              ]
            }
          }
        }
      }
    }
  }
}
```

`maximumError` causes `ng build --configuration=production` to exit non-zero — CI just needs to run that build and check the exit code, no extra tooling required for the hard gate.

2. **Add a PR-level size-diff report** so developers see *why* it grew, not just that it did. Use `source-map-explorer` or `esbuild`'s built-in `--metafile` (Angular's new esbuild-based builder emits stats) to diff against the base branch's bundle:

```yaml
jobs:
  bundle-budget:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - name: Build with stats
        run: npx ng build --configuration=production --stats-json
      - name: Compare against main
        run: |
          npx bundlesize2 --config bundlesize.config.json
      - name: Comment size diff on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('dist/stats.json', 'utf8');
            // parse and post a comparison comment
```

3. **Use `--configuration=production` budgets as the merge-blocking check**, and a separate `source-map-explorer` visualization uploaded as an artifact for humans to drill into which dependency (a moment.js import, an unused Material module, a duplicated lodash) caused the jump. This separates "gate" (binary pass/fail, deterministic) from "diagnosis" (rich, human-consumed).

4. **Guard against silent budget creep via ratcheting**, not just a fixed ceiling: store the current bundle size in a checked-in `.size-budget.json` after each merge to `main`, and have CI fail if a PR's size exceeds `previous + small-tolerance` (e.g., 2KB) even if it's still under the absolute `maximumError`. This catches death-by-a-thousand-cuts growth that a static 600KB ceiling wouldn't catch until it's too late.

5. **Common root causes to call out:** barrel-file re-exports pulling in whole modules via tree-shaking failures, accidentally importing a dev-only library in production code, moment.js/lodash instead of date-fns/lodash-es, and not using `@defer` blocks (Angular's deferred-loading control flow) for below-the-fold heavy components.

```html
@defer (on viewport) {
  <heavy-chart [data]="data" />
} @placeholder {
  <div class="chart-skeleton"></div>
}
```

**Interviewer intent:** Tests whether the candidate knows Angular's native budget mechanism (not just "add a webpack-bundle-analyzer step") and can distinguish a hard CI gate from a diagnostic report.

---

## Scenario 4: Visual regression testing for a shared design-system library

**Situation:** Your company publishes an internal Angular component library (buttons, cards, form fields) consumed by 12 product teams. A recent PR that "only changed a variable name" in the SCSS accidentally shifted padding on every button across all consuming apps, discovered two days after release by an angry team lead. Unit tests didn't catch it because they don't assert on rendered pixels.

**Question:** Design a visual regression testing (VRT) strategy for this library that runs in CI on every PR.

**Answer:**
Unit/component tests validate behavior and DOM structure; they cannot catch "this now renders 4px lower." You need pixel-level snapshot comparison, which means: render each component in isolation (Storybook is the natural harness for a design system), capture screenshots, and diff against approved baselines.

1. **Use Storybook + Chromatic (or Loki/Percy/BackstopJS if you must self-host) as the harness.** Storybook gives you a stable, isolated render target per component/state — exactly the granularity you need for a design system (every variant: primary/secondary button, disabled state, RTL, dark mode).

```ts
// button.stories.ts
export const AllVariants: Story = {
  render: () => ({
    template: `
      <div style="display:flex; gap:8px;">
        <lib-button variant="primary">Primary</lib-button>
        <lib-button variant="secondary">Secondary</lib-button>
        <lib-button variant="primary" [disabled]="true">Disabled</lib-button>
      </div>
    `,
  }),
};
```

2. **Wire VRT into CI as a required check, but with a human-in-the-loop approval step for legitimate changes** — VRT differs from unit tests in that a "failure" isn't always a bug; it might be an intentional visual change that needs design sign-off.

```yaml
jobs:
  visual-regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx nx run design-system:build-storybook
      - name: Run Chromatic
        uses: chromaui/action@v11
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          buildScriptName: build-storybook
          exitOnceUploaded: true
          onlyChanged: true   # TurboSnap: only snapshot stories affected by the diff
```

3. **Use TurboSnap / affected-story detection** so VRT doesn't re-render and diff all 400 stories on every PR — only stories whose component (or transitive dependency) changed, mirroring the Nx-affected philosophy from Scenario 1 but at the visual layer.

4. **Baseline approval workflow:** a visual diff blocks merge until a design-system maintainer explicitly "accepts" the new baseline in the Chromatic UI. This converts an accidental padding regression into a required, visible decision rather than a silent merge — exactly the failure mode in the situation.

5. **Cross-browser and theming matrix**, because CSS bugs are often browser- or theme-specific:

```yaml
strategy:
  matrix:
    browser: [chromium, firefox, webkit]
    theme: [light, dark]
```

6. **Self-hosted alternative (if no budget for Chromatic):** Playwright's built-in `toHaveScreenshot()` with baseline images committed to the repo, diffed via pixelmatch, run in a fixed Docker image (font rendering differs across OSes, so CI must render screenshots in the exact same container image every time, never on developer machines, or diffs will be pure noise from font antialiasing).

```ts
test('primary button renders correctly', async ({ page }) => {
  await page.goto('/iframe.html?id=button--primary');
  await expect(page.locator('lib-button')).toHaveScreenshot('button-primary.png', {
    maxDiffPixelRatio: 0.01,
  });
});
```

**Interviewer intent:** Tests awareness that unit tests and visual tests catch fundamentally different classes of bugs, and whether the candidate knows to gate on human review rather than treating pixel diffs as pass/fail like a unit test.

---

## Scenario 5: Preview deployments per pull request

**Situation:** Product managers and designers currently review UI changes by pulling the branch locally and running `ng serve`, which is slow and error-prone (wrong Node version, stale `node_modules`). Leadership wants every PR to automatically get a live, shareable preview URL.

**Question:** Design a per-PR preview deployment pipeline for an Angular SPA, including how it handles environment configuration, cleanup, and multiple concurrent PRs.

**Answer:**
Per-PR previews require three things: a build parameterized correctly for a non-production, non-localhost base path/API target; a unique deployment target per PR; and automatic teardown so previews don't pile up forever.

1. **Parameterize the Angular build for preview via file replacement or environment injection** — do not hardcode a preview API URL into `environment.prod.ts`; instead inject it at build/runtime.

```json
// angular.json — a preview configuration
"configurations": {
  "preview": {
    "fileReplacements": [
      { "replace": "src/environments/environment.ts", "with": "src/environments/environment.preview.ts" }
    ],
    "outputHashing": "all"
  }
}
```

For fully dynamic per-PR API targets, prefer a runtime-loaded `config.json` fetched by `APP_INITIALIZER`/`provideAppInitializer` rather than a build-time constant, so the same artifact can point at different backends without rebuilding:

```ts
export function initConfig(configService: ConfigService) {
  return () => configService.loadConfig(); // fetches /assets/config.json at runtime
}
```

2. **Deploy to a PR-scoped path or subdomain**, using the PR number as the unique key:

```yaml
name: Preview Deploy
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - run: npm ci
      - run: npx ng build --configuration=preview
      - name: Deploy to Netlify (alias per PR)
        uses: nwtgck/actions-netlify@v3
        with:
          publish-dir: dist/app/browser
          alias: pr-${{ github.event.pull_request.number }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Preview deployed: https://pr-${context.issue.number}--myapp.netlify.app`
            });
```
(Equivalent patterns exist for Firebase Hosting preview channels, Vercel, AWS Amplify, or a self-hosted S3+CloudFront-per-prefix setup fronted by a Lambda@Edge router.)

3. **Handle routing for SPA deep links** — since previews live at a path/subdomain, make sure the Angular router's `APP_BASE_HREF` and the hosting platform's rewrite rules (`/* -> /index.html`) are consistent, or refreshing a deep-linked preview route 404s.

4. **Automatic teardown on PR close**, so previews don't accumulate indefinitely (cost + a stale environment giving false confidence weeks later):

```yaml
on:
  pull_request:
    types: [closed]
jobs:
  teardown:
    runs-on: ubuntu-latest
    steps:
      - name: Delete Netlify alias
        run: netlify api deleteSiteDeploy --data '{"deploy_id": "..."}'
```

5. **Isolate backend state per preview if the app is full-stack** — if the preview needs a live API, either point it at a shared staging backend (simplest, but concurrent PRs can clobber each other's test data) or spin up an ephemeral backend per PR (e.g., a Neon/PlanetScale branched database + a preview API container) — call out this tradeoff explicitly, since it's usually the crux of a real design discussion.

6. **Concurrency control** — use GitHub Actions' `concurrency` group keyed by PR number so pushing new commits cancels the in-flight preview build for the same PR instead of racing two deploys:

```yaml
concurrency:
  group: preview-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

**Interviewer intent:** Tests whether the candidate thinks about the full lifecycle (create, update, teardown) and backend/environment isolation, not just "run ng build and upload it somewhere."

---

## Scenario 6: Parallelizing a 50-minute Jest/Karma unit test suite

**Situation:** The unit test suite has grown to 8,000 specs and takes 50 minutes serially in CI, becoming the single largest contributor to PR turnaround time. The team has already migrated from Karma to Jest for faster startup, but the wall-clock time hasn't improved much because everything still runs in one job.

**Question:** How would you parallelize this suite across CI runners, and what pitfalls would you watch for?

**Answer:**
Parallelization has two independent levers: **sharding across CI machines** (distribute test files across N runners) and **worker concurrency within a machine** (Jest's `--maxWorkers`). Both matter, and naive approaches to either can produce misleading speedups or subtly broken tests.

1. **Shard at the CI-matrix level using Jest's built-in `--shard` flag** (Jest 28+), which splits the *test file* list (not individual tests) evenly across N declared shards:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - run: npm ci
      - run: npx jest --shard=${{ matrix.shard }}/4 --ci --coverage --coverageReporters=json
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.shard }}
          path: coverage/coverage-final.json

  merge-coverage:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { pattern: 'coverage-*', merge-multiple: true }
      - run: npx nyc merge . merged-coverage.json && npx nyc report --reporter=lcov
```

2. **Use timing-based sharding, not naive alphabetical splitting**, if your suite has wildly uneven file sizes (one 400-test integration-style spec vs. 200 tiny unit specs) — otherwise one shard finishes in 2 minutes and another takes 20, and your wall-clock time is bound by the slowest shard. Jest's `--shard` alone splits by count, not historical duration; pair it with a test-timing report (`--json` output from a previous run) fed into a custom bucketing script, or use Nx's `nx affected -t test --parallel` combined with Nx Cloud's Distributed Task Execution, which does timing-aware bin-packing automatically across agents.

3. **Combine `--maxWorkers` (in-machine parallelism) with shard count deliberately** — on a 4-core GitHub-hosted runner, `--maxWorkers=4` per shard, times 4 shards, means you're using 16 logical cores' worth of parallelism across the fleet; oversubscribing a single runner's cores (`--maxWorkers=100%` on a 2-core box) causes context-switch thrashing that makes tests *slower*, not faster.

```bash
npx jest --shard=2/4 --maxWorkers=2 --ci
```

4. **Watch for cross-test pollution that only manifests under parallelism**: tests that mutate a shared singleton (Angular's `TestBed` module cache), rely on execution order, or write to the same temp file/port will pass serially and fail randomly when sharded — because Jest runs each file in its own worker process/module registry by default, but *within* a file, `TestBed.configureTestingModule` state leaking between `it` blocks without `TestBed.resetTestingModule()` in `afterEach` is a classic silent-order-dependency bug that sharding exposes.

```ts
afterEach(() => {
  TestBed.resetTestingModule();
});
```

5. **Fail fast vs. full report tradeoff**: `--bail` stops on first failure for faster feedback on a broken build, but for PR CI you generally want `--ci` (no bail) so a single failing shard doesn't hide other real failures in different shards — developers should see the full failure picture in one push, not discover a second bug after fixing the first.

6. **Coverage merging is required** because coverage runs are now split across processes — using `nyc merge` (as above) or Jest's `--coverageReporters=json` per shard plus a final merge step ensures your coverage threshold check operates on the *whole* suite's coverage, not one shard's partial view (a naive per-shard coverage gate would incorrectly fail every shard for "low coverage" since each only sees a quarter of the code paths).

**Interviewer intent:** Tests understanding of the difference between file-level sharding and worker-level concurrency, and awareness that parallelism surfaces test isolation bugs that were previously hidden by accidental ordering.

---

## Scenario 7: Automating semantic versioning and changelogs for a published component library

**Situation:** Your team publishes `@acme/ui-kit` to a private npm registry, consumed by multiple internal apps. Currently, versioning and changelog writing are manual: someone remembers to bump `package.json`, forgets half the time, and the changelog is perpetually out of date, causing consuming teams to be surprised by breaking changes.

**Question:** Design an automated release pipeline that determines the version bump and changelog from commit history, and publishes on merge to main.

**Answer:**
The standard, tool-agnostic approach is **Conventional Commits + semantic-release** (or Nx's built-in release capabilities if already on Nx, or Changesets if you want per-PR explicit intent rather than commit-message parsing). The key design decision is: where does "this is a breaking change" get declared, and how deterministically can CI derive a version from it?

1. **Enforce Conventional Commits at commit time**, not just at release time, so the signal (`feat:`, `fix:`, `BREAKING CHANGE:`) is reliably present in history. Use `commitlint` in a git hook and/or a CI check on PR titles (if you squash-merge, the PR title *is* the commit message, so validate the PR title instead):

```yaml
# .github/workflows/lint-pr.yml
jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

2. **Use semantic-release to compute the version bump and changelog on merge to main**, driven entirely by commit type: `fix:` → patch, `feat:` → minor, `BREAKING CHANGE:` footer or `!` → major.

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    ["@semantic-release/npm", { "npmPublish": true, "pkgRoot": "dist/ui-kit" }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }],
    "@semantic-release/github"
  ]
}
```

```yaml
jobs:
  release:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write   # for npm provenance
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, registry-url: 'https://registry.npmjs.org' }
      - run: npm ci
      - run: npx nx build ui-kit --configuration=production
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

3. **If the library lives inside an Nx monorepo alongside apps**, prefer `nx release` (Nx's first-party versioning tool), which understands the project graph and can version independently or in lockstep, and correctly bumps *dependent* library versions when a shared dependency changes:

```jsonc
// nx.json
{
  "release": {
    "projects": ["ui-kit"],
    "version": { "conventionalCommits": true },
    "changelog": { "workspaceChangelog": false, "projectChangelogs": true }
  }
}
```

```bash
npx nx release --dry-run   # verify computed version bump before trusting CI
npx nx release
```

4. **Publish npm provenance** (`--provenance` / `id-token: write` permission) so consuming teams can verify the published package was built by this exact CI pipeline from this exact commit, not tampered with — increasingly a supply-chain security expectation, not just a nice-to-have.

5. **Handle pre-release/beta channels** for consumers who want to test breaking changes early, via a `next` branch mapped to a `next` npm dist-tag:

```json
{ "branches": ["main", { "name": "next", "prerelease": true }] }
```

6. **Tradeoff discussion:** commit-message-driven versioning (semantic-release) is fully automatic but fragile if commit hygiene slips (a `feat:` commit that should've been `feat!:` silently ships a breaking change as a minor version). Changesets-style tooling (explicit `.changeset/*.md` files authored per-PR) is more manual but forces an explicit human decision about severity at PR time — many teams prefer Changesets specifically for *libraries* for this reason, reserving semantic-release for internal services where the cost of a versioning mistake is lower.

**Interviewer intent:** Tests knowledge of Conventional Commits, semantic-release/Nx release mechanics, and whether the candidate can articulate the tradeoff between full automation and explicit human-declared severity.

---

## Scenario 8: "Works on my machine" — CI fails, local passes

**Situation:** A developer's PR passes `ng test` and `ng build` locally but fails in CI with `Cannot find module` errors during the build step, and separately, a date-formatting unit test fails in CI but passes locally. The developer insists "it works for me" and is asking you to just re-run the pipeline.

**Question:** Walk through how you'd systematically diagnose environment-parity issues like this, and what CI configuration prevents them going forward.

**Answer:**
"Works locally, fails in CI" almost always traces to one of a small number of environment-parity gaps: dependency resolution differences, Node/OS/locale/timezone differences, or case-sensitivity differences between filesystems. Debug methodically rather than just re-running.

1. **`Cannot find module` in CI but not locally → dependency lockfile drift.** The most common cause is the developer running `npm install` (which can update `package-lock.json` transitively) while CI runs `npm ci` (which installs *exactly* what's locked and fails/differs if `package.json` and `package-lock.json` are out of sync). Verify:

```bash
# Locally, reproduce CI's exact install behavior
rm -rf node_modules
npm ci
npm run build
```

If this alone reproduces the failure locally, the fix is committing an updated, consistent `package-lock.json` — never let engineers `npm install` and forget to commit the lockfile diff. Enforce this with a CI check:

```yaml
- name: Verify lockfile is in sync
  run: npm ci --dry-run 2>&1 | grep -q "would install" && exit 1 || true
```

2. **Case-sensitivity: macOS/Windows filesystems are case-insensitive by default; Linux CI runners are case-sensitive.** An import like `import { Foo } from './Utils'` resolving locally against a file actually named `utils.ts` works on a developer's Mac/Windows machine but fails on Linux CI with exactly a "Cannot find module" error. Catch this class of bug with a linter rule or a dedicated CI step:

```bash
git ls-files | # cross-reference import paths against exact on-disk casing
```
or simply run the build once inside a Linux container locally (`docker run --rm -v $(pwd):/app -w /app node:20 npm ci && npm run build`) to reproduce before pushing.

3. **The flaky date-formatting test → timezone/locale mismatch.** CI runners typically default to UTC; a developer's laptop is in `America/Los_Angeles` or similar. A test asserting `date.toLocaleDateString()` output or doing date-boundary math (`new Date('2026-07-23').getDate()`) can silently differ by a day depending on timezone.

```ts
// Bad: depends on the runner's local timezone
expect(new Date(timestamp).getHours()).toBe(14);

// Good: pin timezone explicitly in the test, or in Jest config
```

```json
// jest.config.js
{
  "testEnvironmentOptions": { "TZ": "UTC" }
}
```
And make CI and local dev consistent by pinning `TZ=UTC` for both, or better, make the *code* timezone-explicit (use `date-fns-tz`/`Temporal`-style explicit zone handling) rather than relying on ambient environment timezone at all — that's the real fix, since "make CI match my laptop" just relocates the bug rather than removing it.

4. **Node version drift** — ensure `package.json` `engines` field, a committed `.nvmrc`, and the CI workflow's `node-version` all agree, and enforce it:

```json
// package.json
{ "engines": { "node": ">=20.11 <21" } }
```

```yaml
- uses: actions/setup-node@v4
  with:
    node-version-file: '.nvmrc'   # single source of truth, not a hardcoded number
```

5. **General principle to state in the interview:** the fix for "works on my machine" bugs is never "make CI more lenient" — it's making local dev *reproduce CI's constraints* (via Docker, pinned Node/TZ, `npm ci` instead of `npm install`) so the developer's machine stops lying to them. A devcontainer (`.devcontainer/devcontainer.json`) pinned to the same base image as the CI runner is the most robust long-term fix.

**Interviewer intent:** Tests systematic debugging methodology for environment-parity bugs and whether the candidate reaches for root-cause fixes (lockfile discipline, explicit timezone handling) instead of superficial ones (just re-run CI).

---

## Scenario 9: Choosing between GitHub Actions matrix builds and Nx Cloud distributed task execution

**Situation:** Your affected-based pipeline from Scenario 1 works well for small PRs but still takes 18 minutes on PRs that touch a widely-shared library (triggering 30+ affected projects). Leadership asks whether adding more GitHub Actions runners to a matrix would help, or whether you should adopt Nx Cloud's Distributed Task Execution (DTE).

**Question:** Explain the architectural difference between a GitHub Actions matrix and Nx Cloud DTE, and when each is the right tool.

**Answer:**
A GitHub Actions **matrix** is a *static, pre-declared* fan-out: you decide the shard count and assignment at workflow-authoring time (e.g., "always split by 4"), and each matrix job runs the *same* command against a *predetermined* slice. Nx Cloud **DTE** is *dynamic, runtime bin-packing*: an orchestrator (the CI job that calls `nx affected`) distributes individual tasks (this specific project's `build`, that specific project's `test`) across a pool of agents in real time based on the actual dependency graph and historical task duration, re-balancing as tasks complete.

1. **Matrix approach — good for uniform, independent workloads** (e.g., sharding one big Jest suite by file count, as in Scenario 6), but awkward for `nx affected` because the *set* of affected projects varies per PR — a matrix with a fixed shard count of 4 either idles shards on a small PR (only 2 projects affected) or bottlenecks on a huge PR (30 affected projects crammed into 4 shards with naive splitting).

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx nx affected -t build --parallel=3 # each shard still runs the FULL affected set independently unless you manually partition it — wasteful
```

2. **Nx Cloud DTE — purpose-built for graph-shaped, variable workloads.** You declare a pool of agents; the main CI job computes the affected graph and DTE schedules individual project-tasks (respecting dependency order — e.g., `lib-a`'s build must finish before `app-x`'s build that depends on it starts) across whichever agents are free, using historical timing data to bin-pack efficiently.

```yaml
jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@v4
      - run: npm ci
      - uses: nrwl/nx-set-shas@v4
      - run: npx nx-cloud start-ci-run --distribute-on="5 linux-medium-js"
      - run: npx nx affected -t lint test build --parallel=3

  agents:
    runs-on: ubuntu-latest
    strategy:
      matrix: { agent: [1, 2, 3, 4, 5] }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npx nx-cloud start-agent
```

3. **Key tradeoff:** matrix builds are simpler, need no third-party service, and are fully under your control — good default for a suite with roughly uniform, independent test files. DTE requires an external dependency (Nx Cloud account, network calls to its orchestrator) and has a learning curve, but scales far better for monorepos where the *shape* of work changes every PR and tasks have real dependency ordering (you can't just chop `nx affected -t build` into 4 arbitrary buckets when `app-x` depends on `lib-a`'s build output).

4. **Combine both when it makes sense**: use DTE agents provisioned *via* a GitHub Actions matrix (as shown above — the `agents` job matrix just spins up N fungible workers that DTE's scheduler then assigns individual tasks to). This is the common real-world pattern: matrix for infrastructure provisioning, DTE for intelligent task assignment on top of it.

**Interviewer intent:** Tests whether the candidate understands that "more parallel runners" isn't automatically the same as "smarter task distribution," and can reason about static vs. dynamic scheduling tradeoffs.

---

## Scenario 10: A secret leaked into a public build artifact

**Situation:** A security scan discovers that an API key was baked into the production JS bundle of a public Angular app three releases ago, because a developer added `apiKey: 'sk-live-...'` directly to `environment.prod.ts` for "quick testing" and it was never removed before merging. The key has since been used by an attacker.

**Question:** How do you redesign the pipeline to make this class of leak structurally impossible, not just "remind people not to do it"?

**Answer:**
Client-side Angular bundles are, by definition, fully downloadable and inspectable by anyone — so the real fix isn't "prevent leaking secrets from the frontend," it's "ensure no secret can be present in a frontend build at all," backed by CI enforcement, since human review alone already failed here.

1. **Immediate remediation:** rotate the leaked key, and audit git history — since the key was in a committed file, it remains in git history even after being removed from the current `HEAD`, so it must be purged (`git filter-repo` or BFG) or (more realistically for a live key) simply treated as permanently compromised and rotated, with history-scrubbing as defense-in-depth only.

2. **Structural fix #1 — secret scanning as a blocking pre-merge CI gate**, not just a periodic audit:

```yaml
jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Gitleaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
Also enable GitHub's built-in push protection for secret scanning, which blocks the push at the `git push` step itself (before it even reaches CI) for known secret patterns.

3. **Structural fix #2 — any genuinely secret value (private API keys, not public-facing config) simply must never be a build-time constant reachable by the Angular compiler.** Angular's `environment.ts` files are compiled into the client bundle; anything referenced there is public by construction. True secrets belong server-side, accessed via a backend-for-frontend (BFF) proxy that the Angular app calls without ever holding the credential itself.

```ts
// WRONG — compiled into the public bundle
export const environment = {
  production: true,
  stripeSecretKey: 'sk_live_...',
};

// RIGHT — only a public, restricted-scope key (or no key at all; call your BFF)
export const environment = {
  production: true,
  stripePublishableKey: 'pk_live_...',
  apiBaseUrl: 'https://api.acme.com',
};
```

4. **Structural fix #3 — lint rule / CI grep as a second layer**, catching naming-pattern violations even before a secret-scanner's pattern database would (custom internal key formats won't be in gitleaks' default ruleset):

```yaml
- name: Block obvious secret-like env keys
  run: |
    if grep -rE "(secretKey|privateKey|apiSecret)\s*:" src/environments/; then
      echo "Potential secret in environment file"; exit 1
    fi
```

5. **Structural fix #4 — for legitimately sensitive build-time values that differ per environment (not secrets, but things like feature-flag service URLs), inject at deploy time via the runtime-config pattern from Scenario 5** (`config.json` fetched by `APP_INITIALIZER`) rather than baking anything into the compiled bundle, keeping the built artifact identical across environments and secret-free by construction.

6. **Process fix:** treat this as a blameless post-incident review — the developer's shortcut is a symptom of the pipeline not having a gate, not a training failure alone; the fix that prevents recurrence is automated, not a Slack reminder.

**Interviewer intent:** Tests security maturity — whether the candidate understands that client-side bundles are fundamentally public, and reaches for structural/automated prevention (secret scanning gates, BFF pattern) rather than process-only fixes.

---

## Scenario 11: Enforcing code coverage thresholds without gaming the metric

**Situation:** Leadership mandates "80% code coverage or the PR is blocked." Three months later, coverage sits at 81%, but the team has quietly stopped trusting the test suite — developers write trivial tests that execute lines without asserting anything meaningful, just to hit the number.

**Question:** How would you design a coverage-gating strategy that improves real test quality instead of incentivizing coverage theater, and how do you technically implement the gate?

**Answer:**
This is fundamentally a Goodhart's Law problem — "when a measure becomes a target, it ceases to be a good measure." The fix is layering coverage with mutation testing and diff-scoped thresholds, plus not gating on the metric alone.

1. **Technical implementation of a coverage gate** (Jest, as the baseline mechanism):

```json
// jest.config.js
{
  "coverageThreshold": {
    "global": { "branches": 75, "functions": 80, "lines": 80, "statements": 80 }
  }
}
```

```yaml
- run: npx jest --coverage --ci
```

2. **Prefer diff-coverage over global-coverage gating.** A global 80% threshold punishes touching old, under-tested files (a one-line bug fix in a legacy module can fail the whole-repo gate) and, worse, is trivially satisfiable by adding assertion-free tests elsewhere. Gate instead on *coverage of the changed lines in this PR*:

```yaml
- name: Diff coverage check
  run: |
    npx jest --coverage --coverageReporters=json-summary
    npx diff-cover coverage/lcov.info --compare-branch=origin/main --fail-under=85
```
This says "new/changed code must be well-tested" without retroactively demanding legacy code be rewritten, and it can't be gamed by padding unrelated files.

3. **Add mutation testing as the quality backstop coverage can't provide.** Coverage measures "was this line executed," not "would a bug here be caught." Mutation testing (Stryker for Angular/TypeScript) deliberately mutates your source (`>` becomes `>=`, `+` becomes `-`) and checks whether *any* test fails — if not, that "covered" line has no meaningful assertion.

```json
// stryker.conf.json
{
  "testRunner": "jest",
  "mutate": ["src/**/*.ts", "!src/**/*.spec.ts"],
  "thresholds": { "high": 80, "low": 60, "break": 50 }
}
```

```yaml
- run: npx stryker run
```
Run mutation testing less frequently than unit tests (nightly or on a `libs/` affected subset) since it's computationally expensive — not a per-PR blocking gate for the whole repo, but a strong signal for the specific files a PR touches.

4. **Make the review-culture fix explicit, since tooling alone won't fix "assertion-free tests":** add a lightweight PR checklist item / code-review norm that a reviewer explicitly asks "does this test fail if I revert the fix?" — and consider excluding trivial getter/setter/DTO files from the coverage denominator entirely (`collectCoverageFrom` exclusions) so the metric measures meaningful code, not boilerplate.

```json
{
  "collectCoverageFrom": [
    "src/**/*.ts",
    "!src/**/*.module.ts",
    "!src/**/*.dto.ts",
    "!src/**/index.ts"
  ]
}
```

5. **Report trend, not just threshold** — a coverage dashboard (Codecov/Coveralls) showing the trend line over time, with PR comments showing "+2.3% on changed files," gives useful signal without a single brittle pass/fail number being the whole story.

**Interviewer intent:** Tests whether the candidate recognizes Goodhart's Law in a metrics-gating context and knows concrete alternatives (diff-coverage, mutation testing) beyond just "set the threshold higher."

---

## Scenario 12: Multi-environment deployment pipeline with manual approval gates

**Situation:** Your Angular app deploys automatically to a dev environment on every merge to `main`, but staging and production deployments currently happen via someone manually running a deploy script from their laptop — leading to "which version is actually in prod?" confusion and at least one incident where staging config was accidentally deployed to production.

**Question:** Design a promotion-based CD pipeline (dev → staging → production) with proper approval gates and artifact traceability.

**Answer:**
The key principle is **build once, promote the same artifact** — never rebuild per environment (a rebuild risks picking up a different dependency version or a merged commit between environments, which is exactly the "which version is in prod" confusion described).

1. **Build a single, environment-agnostic artifact on merge to `main`**, tagged with the commit SHA, and store it as a versioned build artifact (not rebuilt later):

```yaml
name: Build and Promote
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx ng build --configuration=production
      - uses: actions/upload-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: dist/app/browser
          retention-days: 30

  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist-${{ github.sha }} }
      - run: ./deploy.sh dev dist-${{ github.sha }}

  deploy-staging:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment:
      name: staging
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist-${{ github.sha }} }
      - run: ./deploy.sh staging dist-${{ github.sha }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production   # GitHub Environments: configure "required reviewers" here
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist-${{ github.sha }} }
      - run: ./deploy.sh production dist-${{ github.sha }}
```

2. **Use GitHub Environments' native "required reviewers" protection rule on `production`** (and optionally `staging`) — this pauses the job and requires a named approver to click "Approve" in the Actions UI before the `deploy-production` job runs, giving a real audit trail (who approved, when) instead of a Slack message and a laptop script.

3. **Runtime configuration must differ per environment without rebuilding** — use the `config.json`/`APP_INITIALIZER` runtime-config pattern (Scenario 5) so the *same* built JS bundle behaves correctly against dev/staging/prod APIs; this is precisely what "build once, promote" requires, since Angular's `fileReplacements`-based `environment.ts` approach bakes config into the bundle at build time and would force a rebuild per environment (reintroducing the exact risk that caused the original incident).

4. **Traceability**: tag the deployed commit SHA visibly (e.g., a `/version.json` endpoint the app exposes, or a footer showing `git.sha`), and make every deploy step log to a central deployment audit trail (e.g., a GitHub Deployment status, or a Slack notification with commit link) so "what's in prod right now" is always answerable in seconds.

```ts
// exposed via a static asset generated at build time
// dist/app/browser/version.json → { "sha": "a1b2c3d", "builtAt": "2026-07-23T10:00:00Z" }
```

5. **Rollback plan**: because artifacts are retained and tagged by SHA, rollback is "redeploy the previous SHA's artifact" (fast, no rebuild, no risk of picking up an unintended commit), not "revert and rebuild," which is slower and riskier under incident pressure.

**Interviewer intent:** Tests the "build once, promote everywhere" principle and whether the candidate can design approval gates using platform-native features rather than ad hoc manual processes.

---

## Scenario 13: Slow `npm ci` dominating every pipeline run

**Situation:** Profiling your CI shows that `npm ci` alone takes 3–4 minutes on every job, across every workflow (lint, test, build, e2e) — meaning a PR triggering 5 parallel jobs pays that cost 5 times, adding up to more compute time than the actual work being done.

**Question:** How do you reduce dependency-installation overhead across a pipeline with many parallel jobs?

**Answer:**
Attack this on three fronts: caching the npm/node_modules cache correctly, avoiding redundant installs across jobs via artifact reuse, and reducing what actually needs installing.

1. **Cache `~/.npm` (the download cache) keyed on the lockfile hash** — this is the single biggest lever, since `npm ci` still has to resolve/download packages without it:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: 'npm'   # automatically caches ~/.npm keyed on package-lock.json hash
```
This alone typically cuts `npm ci` from minutes to ~20–40 seconds on a cache hit, since it skips network downloads (though it still re-links `node_modules` from the cache).

2. **Install once, reuse via artifact across the fan-out**, instead of every one of the 5 parallel jobs (lint/test/build/e2e/bundle-check) independently running `npm ci`:

```yaml
jobs:
  install:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - run: npm ci
      - uses: actions/upload-artifact@v4
        with: { name: node-modules, path: node_modules, include-hidden-files: true }

  lint:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: node-modules, path: node_modules }
      - run: npx nx affected -t lint

  test:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: node-modules, path: node_modules }
      - run: npx nx affected -t test
```
Tradeoff: uploading/downloading `node_modules` as an artifact has its own I/O cost (it can be a large directory) — measure whether artifact transfer time beats a per-job cache-hit `npm ci`; for very large `node_modules`, the npm cache approach (option 1) alone is often sufficient and simpler, avoiding this extra complexity.

3. **Use `npm ci --prefer-offline` and consider a faster package manager** if lockfile-cache-hit installs are still slow — pnpm's content-addressable store and hard-linking approach is typically 2–3x faster than npm for cache-hit installs in monorepos with many shared dependencies, though migrating package managers is a bigger decision with its own workspace/lockfile migration cost.

4. **Use a self-hosted runner or a warm container image with `node_modules` pre-baked** for very install-sensitive pipelines — build a custom CI base Docker image that already contains the current lockfile's `node_modules`, rebuilt only when the lockfile changes (via an image-build workflow triggered on `package-lock.json` changes), so PR jobs `docker run` against an image where dependencies are already present, skipping install entirely for the common case.

5. **Reduce what's installed**: audit whether `devDependencies` needed only for local tooling (e.g., a heavy IDE-integration package) are actually required in CI, and use `npm ci --omit=dev` for jobs that only need production dependencies (e.g., a deploy-artifact-packaging job that doesn't run tests).

**Interviewer intent:** Tests practical CI performance-tuning skills — caching strategy, avoiding redundant work across a fan-out, and awareness of tradeoffs between caching approaches rather than reaching for one "silver bullet" fix.

---

## Scenario 14: Zoneless Angular app breaking existing E2E waits in CI

**Situation:** Your team migrated an Angular app to zoneless change detection (`provideExperimentalZonelessChangeDetection` / the stable zoneless API in recent Angular versions) for performance. After the migration, the Playwright E2E suite — previously reliable — now has widespread flakiness in CI: assertions run before the DOM has updated after an async action.

**Question:** Why does removing Zone.js break existing E2E wait strategies, and how do you fix the test suite?

**Answer:**
Many E2E/testing utilities (older Protractor-style patterns, and some custom Playwright helpers) implicitly relied on Zone.js's global "app is stable" signal (`NgZone.isStable` / the `onStable` observable) to know when to proceed — Zone.js patches async APIs (`setTimeout`, `Promise`, DOM events) so Angular can track "is there pending async work," and tools plugged into that signal to wait for stability before asserting. Zoneless Angular has no such global patch, so any wait strategy depending on `NgZone` stability polling silently stops working (it either always says "stable" immediately, or the API isn't wired up at all), and assertions race ahead of rendering.

1. **Diagnosis first: confirm this is the actual cause**, not unrelated flakiness — check whether any test helper calls into `browser.waitForAngular()`-style APIs or a custom `waitForNgStability` utility built on `NgZone`. In a zoneless app these become no-ops.

```ts
// This kind of helper is now meaningless post-zoneless:
async function waitForAngularStable(page: Page) {
  await page.evaluate(() => (window as any).ng?.getComponent && ...);
  // relies on Zone-based stability signals that no longer exist
}
```

2. **Fix: stop waiting on framework-internal signals; wait on observable UI state instead** — this is actually the *correct* Playwright pattern regardless of zoneless, and the migration is a forcing function to fix tests that were relying on an implementation detail:

```ts
// Bad (implicit, framework-coupled)
await page.click('#submit');
await waitForAngularStable(page);
expect(await page.textContent('#result')).toBe('Success');

// Good (explicit, resilient to zoneless AND zone-based apps)
await page.click('#submit');
await expect(page.locator('#result')).toHaveText('Success');   // auto-retrying assertion
```
Playwright's `expect(locator).toHaveText()`/`toBeVisible()` etc. auto-retry internally for a configurable timeout, polling the actual DOM rather than a framework stability flag — this is strictly more robust because it doesn't care *why* the DOM hasn't updated yet (pending HTTP call, `setTimeout`, signal update batching), it just waits for the observable truth.

3. **For component-level (TestBed) tests, zoneless requires explicit change-detection triggering** since there's no automatic zone-triggered CD after async operations:

```ts
it('updates the view after data loads', async () => {
  const fixture = TestBed.createComponent(MyComponent);
  fixture.detectChanges();
  await fixture.whenStable();      // waits for pending async (signals, effects) to settle
  fixture.detectChanges();
  expect(fixture.nativeElement.textContent).toContain('Loaded');
});
```
`ComponentFixture.whenStable()` still works correctly in zoneless mode (it's built on Angular's internal scheduler/`ApplicationRef.isStable`, which zoneless apps still expose via signals/microtask-based tracking) — but tests that relied on `fixture.detectChanges()` being implicitly triggered by *any* async completion (a zone-based-only convenience) must now call it explicitly after the awaited work.

4. **CI-specific consideration**: since this bug is timing-dependent, it may pass locally (fast machine, DOM updates before the next assertion line executes by coincidence) and fail in CI (slower/throttled runners expose the race) — exactly the Scenario 8 pattern, reinforcing why explicit auto-retrying waits matter more, not less, under CI's resource constraints.

5. **Broader lesson to state explicitly**: any test infrastructure that reaches into framework internals (Zone patching, `NgZone.isStable`, private component internals via `ng.probe`) is inherently fragile across Angular version/architecture changes; testing against the rendered DOM and public component API is what survives migrations like zoneless, standalone components, or SSR hydration changes.

**Interviewer intent:** Tests deep understanding of Zone.js's actual role (async tracking, not just "change detection magic") and whether the candidate can connect a framework architecture change to concrete test-infrastructure breakage — a genuinely current (recent Angular versions) scenario.

---

## Scenario 15: SSR/hydration build producing a working dev build but a broken CI-built production bundle

**Situation:** Your Angular Universal (SSR) app renders correctly with `ng serve` and even with a local `ng build && node dist/server/server.mjs`. But the CI-built and deployed version throws `NG0500: Hydration expects the DOM structure to match` errors in production, and users briefly see a flash of unstyled/re-rendered content.

**Question:** What CI/build pipeline differences commonly cause SSR hydration mismatches that don't show up locally, and how do you catch them before deploy?

**Answer:**
Hydration mismatches happen when the server-rendered DOM and the client's first-render DOM disagree — and several common causes are specifically tied to *environment* differences between local dev and the CI-built production artifact, not the code itself.

1. **Different environment variables/config between the SSR build step and the runtime environment** — if the CI pipeline builds with one API endpoint (or feature flag state) and the deployed server runtime resolves a different one (e.g., a `NODE_ENV`-gated code path, or a runtime `config.json` fetched differently on server vs. client), the server renders one DOM shape and the client hydrates expecting another. Verify build-time and runtime config are the *same* values in the CI-produced artifact as in the deployed runtime:

```yaml
- name: Build SSR bundle
  run: npx ng build --configuration=production
  env:
    NODE_ENV: production   # must match the deployed runtime's NODE_ENV exactly
```

2. **Non-deterministic content rendered differently server vs. client** — the classic culprits are `Date.now()`, `Math.random()`, or locale-dependent formatting (`Intl.DateTimeFormat` without an explicit locale/timezone) executed in a component's render path. Locally, your dev server and your browser are often in the *same* timezone/locale, masking the bug; in CI-deployed production, the server (often UTC, in a US-region container) and the client browser (in the user's local timezone) genuinely differ, and hydration fails structurally.

```ts
// Causes hydration mismatch: server renders one string, client expects another on hydration
{{ lastUpdated | date:'short' }}   // date pipe uses server's locale/TZ at SSR time

// Fix: make locale/timezone explicit and identical regardless of where it executes
{{ lastUpdated | date:'short':'UTC':'en-US' }}
```

3. **Add a CI step that actually runs the SSR server and diffs server-rendered HTML against a headless-browser post-hydration DOM snapshot**, catching this class of bug before deploy rather than in production:

```yaml
- name: Build and start SSR server
  run: |
    npx ng build --configuration=production
    node dist/app/server/server.mjs &
    npx wait-on http://localhost:4000
- name: Check for hydration errors
  run: |
    npx playwright test hydration.spec.ts
```

```ts
// hydration.spec.ts
test('no hydration errors on key routes', async ({ page }) => {
  const errors: string[] = [];
  page.on('console', (msg) => {
    if (msg.text().includes('NG0500') || msg.text().includes('hydration')) {
      errors.push(msg.text());
    }
  });
  await page.goto('http://localhost:4000/dashboard');
  await page.waitForLoadState('networkidle');
  expect(errors).toEqual([]);
});
```

4. **Why this doesn't show up with `ng serve` locally**: `ng serve` typically serves client-side rendering only (no SSR) unless you're specifically running the SSR dev server, so a developer's usual local workflow never actually exercises the server-render-then-hydrate path at all — it's easy to believe "it works locally" while never having tested the actual SSR code path. Ensure local dev has an equivalent SSR-serving script (`npm run serve:ssr`) that developers are expected to run before pushing SSR-affecting changes, and make CI run that exact path as a required check, not an optional one.

5. **Container/runtime parity**: verify the production Docker image's Node version and ICU data (`Intl` behavior can differ between "full-icu" and "small-icu" Node builds) match what's used in CI's SSR verification step — a Node build without full ICU data can format dates/numbers differently, again causing hydration text mismatches that only manifest in the deployed container, not a developer's full-featured local Node install.

**Interviewer intent:** Tests up-to-date knowledge of Angular's modern hydration mechanics (NG0500, deterministic rendering requirements) and the discipline to add CI verification for a code path (SSR) that standard local dev workflows often skip entirely.

---

## Scenario 16: Rate-limited/expensive third-party API calls breaking CI reliability

**Situation:** Your E2E and integration tests call a real third-party payment-provider sandbox API. CI has started failing intermittently with 429 (rate limit) responses, especially during high-traffic hours when multiple PRs run concurrently, and the sandbox account has a hard monthly call quota the team recently exceeded.

**Question:** How do you redesign test infrastructure so external API dependencies don't compromise CI reliability?

**Answer:**
The general principle: CI should never depend on the availability, rate limits, or quota of a third-party service for its core signal — external dependencies belong behind a contract boundary that can be mocked/stubbed for the vast majority of runs, with real integration calls isolated to a small, controlled subset.

1. **Introduce a service-virtualization layer (WireMock, Mock Service Worker, or a custom contract-test double) that the app talks to in CI by default**, recording real API responses once and replaying them deterministically:

```ts
// msw handler recorded from a real sandbox response, replayed in CI
export const handlers = [
  http.post('https://api.paymentprovider.com/v1/charges', () => {
    return HttpResponse.json({ id: 'ch_test123', status: 'succeeded' }, { status: 200 });
  }),
];
```

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - run: npx playwright test   # app configured to route external calls through MSW/mock server, no real network calls, no rate limits, no quota consumption
```

2. **Reserve real sandbox-API calls for a small, separate, non-blocking "contract verification" suite** that runs on a schedule (nightly) rather than per-PR, specifically to catch when the third-party's real API has drifted from your recorded mocks (the classic risk of over-mocking: your mocks silently go stale and no longer represent reality).

```yaml
name: Nightly Contract Verification
on:
  schedule:
    - cron: '0 3 * * *'
jobs:
  contract-test:
    runs-on: ubuntu-latest
    steps:
      - run: npx playwright test --grep @contract   # small tagged subset hitting the real sandbox
        env:
          PAYMENT_API_MODE: live-sandbox
```

3. **Add exponential backoff and request throttling specifically for the contract suite** (since it does hit the real, rate-limited API), rather than for the main suite (which shouldn't need it at all once mocked):

```ts
async function callWithBackoff(fn: () => Promise<Response>, retries = 3): Promise<Response> {
  for (let i = 0; i < retries; i++) {
    const res = await fn();
    if (res.status !== 429) return res;
    await new Promise((r) => setTimeout(r, 2 ** i * 1000));
  }
  throw new Error('Rate limited after retries');
}
```

4. **Use a shared, rate-limit-aware test API key/quota pool with concurrency control** for whichever tests must genuinely hit the real service — a `concurrency` group ensures only one contract-verification run executes at a time cluster-wide, regardless of how many PRs are open, protecting the shared quota:

```yaml
concurrency:
  group: payment-contract-tests
  cancel-in-progress: false   # queue, don't cancel — these are scheduled, not per-PR
```

5. **Contract testing discipline (e.g., Pact) as the principled middle ground** — rather than ad hoc recorded mocks going stale silently, a consumer-driven contract test suite defines the expected request/response shape as a formal contract, verified independently against the provider (ideally by the provider's own CI, if they support Pact Broker) — catching drift systematically instead of "we happened to notice production broke."

6. **Cost/reliability tradeoff to state explicitly**: full mocking maximizes CI reliability and speed but risks silent drift from reality; full live-API testing maximizes fidelity but couples your CI's reliability (and cost) to a third party you don't control. The right answer is layering both, weighted toward mocks for the frequent/blocking path and real calls for infrequent/non-blocking verification.

**Interviewer intent:** Tests architectural judgment about test isolation boundaries and whether the candidate recognizes that "flaky CI due to external APIs" is a design problem (missing a mocking boundary), not a retry-configuration problem.

---

## Scenario 17: Migrating a legacy Jenkins pipeline to GitHub Actions without downtime

**Situation:** Your Angular monorepo's CI has run on a self-hosted Jenkins server for years, with a complex Groovy pipeline (`Jenkinsfile`) handling build, test, artifact signing, and deployment to on-prem servers. Leadership wants to move to GitHub Actions for better GitHub integration and lower maintenance burden, but cannot tolerate a "big bang" cutover that risks breaking deploys.

**Question:** How would you plan and execute this migration incrementally and safely?

**Answer:**
Treat this like any high-risk system migration: run both pipelines in parallel (shadow mode) before cutting over, migrate in stages by pipeline stage rather than all at once, and keep an explicit, tested rollback path to Jenkins until GitHub Actions has proven itself under real load.

1. **Stage 1 — shadow mode: GitHub Actions runs alongside Jenkins, non-blocking, purely for comparison.** Jenkins remains the source of truth (its status is the required check); GitHub Actions runs the equivalent pipeline and its results are only *compared*, never gating.

```yaml
name: CI (shadow - GH Actions)
on: [pull_request]
jobs:
  build-test:
    runs-on: ubuntu-latest
    # NOT set as a required status check in branch protection yet
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx nx affected -t lint test build
```
Run both for several weeks; track divergence (does GH Actions ever pass when Jenkins fails, or vice versa?) and investigate every mismatch — these reveal environment differences (different Node version, different `NODE_OPTIONS`, different available system libraries on self-hosted vs. GitHub-hosted runners) that must be resolved before cutover.

2. **Stage 2 — migrate low-risk, easily-reversible stages first**: lint and unit test are safe to cut over early (failure just means a red PR, no real-world consequence). Build artifact signing and production deployment are the highest-risk stages — migrate those last, only once confidence is high.

3. **Stage 3 — flip the required status check** once shadow-mode has shown consistent parity for an agreed bake-in period, but keep the Jenkins pipeline defined (not deleted) and easily re-enablable for a further defined window, in case an edge case surfaces under real merge-queue load that shadow mode didn't catch (e.g., concurrency/locking behavior around a shared on-prem deployment target).

```yaml
# branch protection rule updated: required check is now "build-test (GH Actions)"
# Jenkins job retained, triggered manually if needed as a fallback
```

4. **Handle on-prem deployment access carefully** — GitHub-hosted runners can't reach on-prem servers directly; this typically requires either GitHub's self-hosted runners registered inside the same network Jenkins had access to, or a bastion/deploy-proxy service. Migrate credentials to GitHub Actions secrets (or better, OIDC-based short-lived credentials if the deployment target supports it) rather than copying long-lived Jenkins credentials verbatim — a migration is a natural opportunity to reduce standing credential exposure.

```yaml
jobs:
  deploy:
    runs-on: self-hosted   # runner registered inside the on-prem network
    steps:
      - run: ./deploy.sh production
```

5. **Rewrite Groovy pipeline logic as composable GitHub Actions**, not a monolithic script — the Jenkinsfile likely has shared logic (e.g., a "notify Slack on failure" step reused across jobs); port this as a reusable workflow or composite action so the new pipeline isn't just a shell-script transliteration but takes advantage of GH Actions' native composition:

```yaml
# .github/actions/notify-slack/action.yml
name: 'Notify Slack'
runs:
  using: 'composite'
  steps:
    - shell: bash
      run: curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"${{ inputs.message }}\"}" ${{ env.SLACK_WEBHOOK }}
```

6. **Communication and rollback plan**: announce the cutover date to all engineers, keep Jenkins in read-only "break glass" mode for at least one full release cycle post-cutover, and define an explicit rollback runbook (re-enable Jenkins as required check, disable GH Actions required check) so an incident during the bake-in period has a fast, rehearsed way back.

**Interviewer intent:** Tests migration planning maturity — shadow-mode validation, staged risk-ordered cutover, and credential-hygiene awareness — rather than just knowing GitHub Actions YAML syntax.

---

## Scenario 18: Monorepo CI where one team's broken `main` blocks every other team

**Situation:** Your Nx monorepo has a shared `main` branch used by 6 teams. Team A merges a change with a subtly broken shared library; because it passed affected-CI (the specific bug only manifests under a runtime condition CI doesn't exercise), `main` is now broken for anyone who bases a new branch off it, and Team B's unrelated PR fails CI with a confusing, unrelated-looking error.

**Question:** How do you both prevent broken `main` in the first place, and reduce the blast radius/confusion when it happens anyway?

**Answer:**
Two separate concerns: prevention (never let a bad merge land on `main` at all) and containment (make it obvious and fast to detect/revert when it happens despite prevention).

1. **Prevention — use a merge queue instead of "test on the PR branch, merge in whatever order humans click merge."** A merge queue (GitHub's native Merge Queue, or Bors/Mergify) re-tests each PR's changes *rebased on top of the latest `main` plus any other queued PRs ahead of it*, immediately before actually merging — catching the specific class of bug where two independently-fine PRs conflict or interact badly when combined, which per-PR CI against a slightly-stale `main` cannot catch.

```yaml
# branch protection: require merge queue
# .github/workflows/merge-queue.yml
on:
  merge_group:
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { ref: ${{ github.event.merge_group.head_sha }} }
      - run: npm ci
      - uses: nrwl/nx-set-shas@v4
      - run: npx nx affected -t lint test build
```

2. **Prevention — a required, fast "main-health" smoke pipeline that runs immediately post-merge**, not just pre-merge, catching gaps between what per-PR affected-CI exercised and what actually breaks on `main` (e.g., a full non-affected build occasionally catching a global config issue affected-scoping missed):

```yaml
on:
  push:
    branches: [main]
jobs:
  post-merge-smoke:
    runs-on: ubuntu-latest
    steps:
      - run: npx nx run-many -t build --all   # full build, not just affected, as a safety net
```

3. **Containment — auto-revert on `main` breakage rather than leaving it broken for hours.** If the post-merge smoke pipeline fails, automatically open a revert PR (many teams wire a bot for this) rather than relying on someone noticing:

```yaml
  auto-revert:
    needs: post-merge-smoke
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          git revert --no-edit ${{ github.sha }}
          git push origin main
      - name: Notify
        run: echo "Reverted broken main commit ${{ github.sha }} — see failing run for details"
```
Pair this with a Slack/Teams alert so the original author is immediately notified and can fix-forward with a new PR, rather than main sitting broken.

4. **Containment — make failure messages point at the actual cause, not a confusing downstream symptom.** Team B's PR failing because of Team A's broken shared library is confusing specifically because the error surfaces in Team B's unrelated build step. Improve this by having the post-merge smoke check (item 2) catch it *before* Team B ever bases work off the broken commit, and by ensuring `nx affected` failure output clearly names which upstream project's build/test actually failed (Nx's default output does this reasonably well — verify your CI log formatting isn't swallowing which specific project failed inside a large parallel run).

5. **Structural fix for "passed CI but broke under a runtime condition CI doesn't exercise":** this points to a CI coverage gap, not a process gap — the fix is adding the missing test case (the specific runtime condition) to the affected library's test suite, and treating "main broke despite green CI" incidents as a required trigger for a coverage retro, not just a revert-and-move-on.

**Interviewer intent:** Tests knowledge of merge queues as a distinct mechanism from plain PR-gate CI, and whether the candidate thinks about blast-radius containment (auto-revert, post-merge smoke tests) as a first-class CI design concern, not just prevention.

---

## Scenario 19: Containerizing the Angular build for reproducibility, and keeping the image small

**Situation:** Your CI currently installs Node and dependencies fresh on a shared runner image that occasionally has subtly different global tool versions (a system-installed Python needed by a native npm module, a different OpenSSL version) causing sporadic, hard-to-reproduce build failures. You decide to containerize the build step for full reproducibility, but need the resulting production image to stay lean for fast deploys.

**Question:** Design a Dockerized, multi-stage build pipeline for an Angular app that's both reproducible in CI and minimal in the final deployed image.

**Answer:**
Use a **multi-stage Dockerfile**: an early stage with the full Node toolchain to install dependencies and run `ng build`, and a final stage that contains only the static output (or SSR server) plus a minimal serving layer — the build tools never ship to production.

1. **Multi-stage Dockerfile for a static SPA:**

```dockerfile
# ---- Stage 1: build ----
FROM node:20.11-bullseye AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx ng build --configuration=production

# ---- Stage 2: serve ----
FROM nginx:1.27-alpine AS runtime
COPY --from=build /app/dist/app/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
The final image contains only nginx + static files — no Node, no `node_modules`, no build toolchain — typically shrinking a 1GB+ build-stage image down to ~40–60MB.

2. **For SSR apps, stage 2 needs Node but still not the dev dependencies:**

```dockerfile
FROM node:20.11-bullseye AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx ng build --configuration=production

FROM node:20.11-bullseye-slim AS runtime
WORKDIR /app
COPY --from=build /app/dist/app/server ./server
COPY --from=build /app/dist/app/browser ./browser
COPY package.json package-lock.json ./
RUN npm ci --omit=dev   # only production deps needed at SSR runtime, not the Angular CLI/build tools
CMD ["node", "server/server.mjs"]
```

3. **Pin exact base image digests, not just tags, for true reproducibility** — `node:20.11-bullseye` can still shift underneath you as the maintainers rebuild that tag; pin by digest for CI images where reproducibility is the explicit goal:

```dockerfile
FROM node:20.11-bullseye@sha256:abcdef1234...
```
Tradeoff: digest-pinning means you must deliberately bump the digest to receive security patches (no automatic drift), which is the correct tradeoff for reproducibility but requires an active update process (e.g., Renovate/Dependabot configured to track Docker digests, not just semver tags).

4. **Build the image itself in CI, not just the app inside it, and cache Docker layers for speed:**

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ghcr.io/acme/app:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```
Layer caching means `npm ci`'s layer is skipped entirely when `package-lock.json` hasn't changed since the last build (Docker's layer cache invalidates only from the first changed instruction onward — hence `COPY package*.json` + `RUN npm ci` *before* `COPY . .` in stage 1 above, so source-code changes don't bust the dependency-install layer).

5. **Scan the final image for vulnerabilities as a CI gate**, since a lean image is also easier to scan quickly:

```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ghcr.io/acme/app:${{ github.sha }}
    exit-code: 1
    severity: CRITICAL,HIGH
```

6. **Why this fixes the original "sporadic failure" problem**: the entire build now happens inside a pinned, identical container image on every run, regardless of which underlying CI runner/VM executes it — the "subtly different global tool versions" root cause is eliminated because nothing outside the container (no host-installed Python, no host OpenSSL) is ever consulted during the build.

**Interviewer intent:** Tests multi-stage Docker proficiency specifically applied to Angular's build/serve split, and whether the candidate reasons correctly about Docker layer caching order and digest-pinning tradeoffs, not just "put it in a container."

---

## Scenario 20: Designing CI checks for library peer-dependency compatibility across multiple Angular major versions

**Situation:** Your team's published Angular component library (`@acme/ui-kit`) supports consumers on Angular v17, v18, and v19 simultaneously via peer dependency ranges. A recent release accidentally broke v17 consumers because the CI only ever tested against the latest Angular version installed in the repo's own `package.json`.

**Question:** Design a CI matrix that verifies compatibility across all supported Angular peer-dependency versions before every release.

**Answer:**
The core idea: CI must actually *install and build/test against* each supported major version in the peer range, not just trust that TypeScript types and `peerDependencies` semver ranges compile correctly in isolation — real breakage often comes from runtime API differences (a deprecated/removed Angular API, a changed default) that only a real build against that version surfaces.

1. **Matrix the CI job across supported Angular majors, each in its own isolated install:**

```yaml
name: Peer Compatibility Matrix
on: [pull_request]

jobs:
  compat:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        angular-version: ['17', '18', '19']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - name: Install pinned Angular major
        run: |
          npm install @angular/core@${{ matrix.angular-version }} \
                      @angular/common@${{ matrix.angular-version }} \
                      @angular/compiler@${{ matrix.angular-version }} \
                      --no-save
      - run: npm ci --ignore-scripts
      - run: npx ng build ui-kit
      - run: npx jest --testPathPattern=projects/ui-kit
```

2. **Maintain a real consumer test-harness app per supported major**, rather than just swapping the library's own dependency versions in place — a thin "smoke consumer" Angular app (one per major version, in separate subfolders or separate CI-only workspaces) that imports and renders every public component from `ui-kit`, catching integration-level breakage (e.g., a component relying on a `NgModule` API removed in a later standalone-only Angular version, or vice versa, an API only available starting a specific version) that testing the library in isolation might miss:

```yaml
  consumer-smoke:
    strategy:
      matrix:
        angular-version: ['17', '18', '19']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          cd compat-harness/ng${{ matrix.angular-version }}
          npm ci
          npm run build
          npm run test
```

3. **Verify `peerDependencies` semver ranges are honestly enforced, not just documented**, using `npm-check-peer-dependencies` or simply asserting the matrix itself is the enforcement — the range in `package.json` should exactly bound what's tested:

```json
{
  "peerDependencies": {
    "@angular/core": ">=17.0.0 <20.0.0",
    "@angular/common": ">=17.0.0 <20.0.0"
  }
}
```
If the matrix only tests 17/18/19 but the range claims `<21.0.0`, that's a gap — the range should never claim support for a version the matrix hasn't actually verified.

4. **Gate releases on the full matrix passing, not just the primary development version:**

```yaml
jobs:
  release:
    needs: [compat, consumer-smoke]   # release blocked unless every supported major passes
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: npx semantic-release
```

5. **Handle version-specific code paths explicitly if APIs genuinely differ across supported majors** (e.g., a signal-based API only available from v17+, or a deprecated `NgModule` pattern still needed for v17 consumers who haven't adopted standalone) — feature-detect at runtime rather than assuming a single code path works everywhere:

```ts
// Defensive feature-detection pattern for cross-version library code
const hasSignalInputs = typeof (ɵɵInput as any)?.required === 'function';
```
Prefer, where possible, sticking to the *lowest common denominator* stable public API across the supported range rather than branching internal logic per version — branching adds real maintenance and test-matrix cost that scales with every additional supported major.

6. **Retire old majors deliberately, on a published support-window policy** (e.g., "we support the last 3 Angular majors, matching Angular's own release cadence"), so the compatibility matrix doesn't grow unboundedly forever — drop v17 from the matrix (and bump the `peerDependencies` floor) on a known schedule communicated to consumers in the changelog, rather than supporting every version indefinitely by accident.

**Interviewer intent:** Tests whether the candidate thinks about library CI as fundamentally different from application CI — needing a compatibility matrix against consumer environments, not just the library's own dev dependencies — a distinction many candidates miss.

---

## Quick Revision Cheat Sheet

- **Affected-only CI** (Nx `affected` / equivalent) scopes builds/tests to what actually changed via the dependency graph, not file diffs — pair with a scheduled full-graph run as a safety net, and with distributed caching (Nx Cloud) keyed on correct input hashes (including env vars).
- **Flaky E2E tests**: quarantine into a non-blocking watch job, never silently increase `retries` as the whole fix — root-cause fixes are per-worker test data isolation, deterministic `waitFor`/auto-retrying assertions instead of hard timeouts, and a tracked flake-rate SLO.
- **Bundle budgets**: enforce with Angular's native `angular.json` `budgets` (`maximumError` fails the build) as the hard merge gate; layer a PR-level size-diff report and ratcheting (fail on regression vs. last-merged size) to catch slow creep the absolute ceiling misses.
- **Visual regression testing** catches a different bug class than unit tests (pixel-level layout regressions) — use Storybook + Chromatic/Percy/Playwright screenshots, gate with human baseline-approval (not pure pass/fail), and scope snapshot runs to affected stories (TurboSnap) for speed.
- **Preview deployments per PR**: build once with runtime-injected config (not baked-in environment files), deploy to a PR-scoped alias, auto-teardown on PR close, and use `concurrency` groups to cancel stale in-flight preview builds.
- **Parallelizing test suites**: combine CI-matrix sharding (`--shard`) with in-machine worker concurrency (`--maxWorkers`), prefer timing-aware bin-packing over naive alphabetical splits, and expect parallelism to expose previously-hidden test-order-dependency bugs (reset `TestBed` state).
- **Semantic versioning/changelog automation**: Conventional Commits + semantic-release (or Nx release) derive version bumps and changelogs automatically from commit history; Changesets trade automation for an explicit human severity decision per PR — pick based on how costly a wrong auto-inferred bump would be.
- **"Works locally, fails in CI"**: check lockfile drift (`npm ci` vs `npm install`), filesystem case-sensitivity (macOS/Windows vs Linux runners), and timezone/locale defaults (UTC in CI vs local machine) — fix by making local dev reproduce CI's constraints (Docker/devcontainer), not by loosening CI.
- **Static matrix vs. dynamic distributed task execution**: a GitHub Actions matrix is a fixed, pre-declared fan-out best for uniform independent workloads; Nx Cloud DTE dynamically bin-packs graph-shaped, dependency-ordered tasks across a runtime agent pool — often combined, matrix for infrastructure, DTE for scheduling.
- **Secrets in client bundles are structurally public** — the fix is never "remind developers," it's blocking secret-scanning CI gates (gitleaks, push protection) plus architecturally keeping true secrets server-side behind a BFF, never in `environment.ts`.
- **Coverage-gating without gaming it**: prefer diff-scoped coverage thresholds over global ones, add mutation testing (Stryker) as a backstop against assertion-free tests, and exclude boilerplate files from the coverage denominator.
- **Multi-environment CD**: build one artifact, promote the identical artifact through dev → staging → production using GitHub Environments' required-reviewer approval gates — never rebuild per environment, since that reintroduces "which version is actually deployed" ambiguity.
- **Slow `npm ci` across many parallel jobs**: cache `~/.npm` keyed on the lockfile hash as the primary lever; consider install-once-reuse-via-artifact or a pre-baked container image for very install-heavy pipelines, weighing artifact-transfer cost against redundant installs.
- **Zoneless Angular breaks Zone-dependent E2E wait helpers**: replace any wait strategy built on `NgZone`/Zone.js stability polling with DOM-state-based auto-retrying assertions (Playwright's `expect(locator)...`), and use `fixture.whenStable()` plus explicit `detectChanges()` in zoneless component tests.
- **SSR hydration mismatches (NG0500) traced to CI/build environment differences**: eliminate non-deterministic rendering (explicit locale/timezone in date pipes, no bare `Date.now()`/`Math.random()` in render paths), and add a CI step that actually runs the SSR server and checks for hydration console errors — since typical `ng serve` local dev often never exercises the real SSR path at all.
- **External API dependencies in tests**: mock/stub by default for the blocking per-PR path (MSW, WireMock, recorded contracts), reserve real API calls for a small non-blocking scheduled contract-verification suite with backoff and concurrency limits — don't let CI reliability depend on a third party's rate limits.
- **Legacy CI platform migration** (Jenkins → GitHub Actions): run new and old in shadow mode before cutover, migrate low-risk stages first, keep a rehearsed rollback path, and use the migration as an opportunity to reduce standing credential exposure (short-lived/OIDC over copied long-lived secrets).
- **Broken `main` in a shared monorepo**: prevent with a merge queue (re-tests each PR rebased on the latest queued state, not a stale `main`) and a post-merge full-build smoke test; contain blast radius with an automated revert-on-failure bot rather than leaving `main` broken until someone notices.
- **Reproducible, lean containerized builds**: multi-stage Dockerfiles keep build toolchains out of the final runtime image, order `COPY package*.json` + `RUN npm ci` before `COPY . .` to preserve Docker layer caching, and pin base images by digest (not just tag) for true reproducibility, accepting the tradeoff of needing active digest-bump tooling for security patches.
- **Library peer-dependency compatibility**: test the actual supported Angular version matrix (real installs of each major, ideally via consumer smoke-test harness apps), gate releases on the full matrix passing, and retire old majors on a published, deliberate support-window schedule rather than by accident.

**Created By - Durgesh Singh**
