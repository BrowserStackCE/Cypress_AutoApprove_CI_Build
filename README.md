# Cypress + Percy Visual Regression (GitHub Actions)

This repository demonstrates fully automated UI and visual regression testing with Cypress and Percy, wired into GitHub Actions for every push and pull request. The pipeline runs Cypress tests in parallel, uploads visual snapshots to Percy, and then automatically approves or rejects the Percy build based on a configurable visual diff threshold.

## Features

- Cypress end‑to‑end tests with Percy visual snapshots.
- Parallel Cypress execution in GitHub Actions for faster feedback.
- Percy build lookup by commit SHA with branch fallback.
- Automatic calculation of visual diff percentage using Percy build stats.
- Auto‑approve or auto‑reject Percy builds in CI, blocking PRs when visual changes exceed a threshold.

## CI Workflow Overview

The main CI workflow lives in `.github/workflows/cypress-percy.yml` and defines two jobs that run on pushes and pull requests (for example, on `main` or `develop`).

### Job 1: `cypress-tests` (parallel)

This job is responsible for running Cypress tests while capturing Percy snapshots.

- Checks out the repository and sets up Node.js.
- Installs project dependencies and the Percy CLI.
- Runs `percy exec` to wrap the Cypress test run so snapshots are sent to Percy.
- Uses multiple containers (for example, matrix strategy with `containers: 2`) so tests and snapshots run in parallel against a single Percy build.

Typical command inside the job:

```bash
npx percy exec -- npx cypress run
```

This creates or updates one Percy build for the commit while each parallel container contributes snapshots to it.

### Job 2: `percy-review`

After the tests finish, a separate job performs an automated review of the Percy build.

- Locates the Percy build for the current commit using the Git SHA, with a fallback search by branch name if needed.
- Polls/retries (for example up to 30 times with a small delay) to allow Percy processing to complete.
- Reads the build statistics, such as total snapshots and total diffs, to compute `diff% = (diffs / total) × 100`.
- Compares `diff%` against a threshold (default `1%`) and then:
  - Marks the build approved when `diff% <= THRESHOLD`.  
  - Leaves the build unapproved and fails the job when `diff% > THRESHOLD`.

Because this job is part of the required PR checks, a failing review job will block the pull request from being merged until the visual differences are addressed.

## Auto‑Approval Threshold

The auto‑approval behavior is controlled by a threshold environment variable inside the workflow, for example:

```bash
THRESHOLD=1
```

- When `diff% <= THRESHOLD`, the script calls Percy’s API or CLI to approve the build and exits successfully.
- When `diff% > THRESHOLD`, the script does not approve the build, exits with a non‑zero code, and causes the GitHub Actions job (and therefore the PR check) to fail.

You can adjust the sensitivity by changing the value of `THRESHOLD` in `.github/workflows/cypress-percy.yml` to a higher or lower percentage depending on how strict your visual QA gate should be.

## Required Environment Variables

Set the following in **GitHub → Repository → Settings → Secrets and variables → Actions**.

| Name        | Required | Description                                  |
|------------|----------|----------------------------------------------|
| `PERCY_TOKEN` | Yes      | Percy project token, used by `percy exec`.  |

If `PERCY_TOKEN` is missing or incorrect, Percy snapshots will not be uploaded and the review job will not find a build for the commit.

## Running Tests Locally

To run the same Cypress + Percy flow on your machine:

1. Install dependencies:

   ```bash
   npm install
   ```

2. Export your Percy project token in the local shell:

   ```bash
   export PERCY_TOKEN=<your-percy-project-token>
   ```

3. Run Cypress through `percy exec`:

   ```bash
   npx percy exec -- npm run test
   ```

This captures Cypress snapshots and uploads them to the same Percy project that CI uses, making it easy to verify visual changes before pushing.[5]

## Visual QA Gate Behavior

| Condition             | Percy Build Status         | CI / PR Result                  |
|----------------------|----------------------------|---------------------------------|
| `diff% <= THRESHOLD` | Build auto‑approved        | Visual check passes, merge allowed. |
| `diff% > THRESHOLD`  | Build left unapproved      | Job fails, merge blocked.       |

If a build is rejected or left unapproved, open the Percy UI, review the snapshot diffs, and either fix the regression or update the baseline when the change is intentional.

## Troubleshooting

- **No Percy build found**  
  - Confirm that `PERCY_TOKEN` is set correctly in GitHub secrets and locally if you run tests on your machine.
  - Check that `percy exec` actually ran around the Cypress command in CI and produced snapshots.
  - Allow some time for Percy to finish processing builds; the review job already retries, but very large builds can take slightly longer.

- **Build rejected or diff% too high**  
  - Inspect diffs in Percy’s web UI to see which components changed.
  - If changes are expected, update the baseline snapshots there; if not, fix the UI and push a new commit.

- **Missing or invalid Percy token**  
  - Ensure `PERCY_TOKEN` exists under repository Actions secrets and matches the project token from Percy’s settings page.

## Useful Links

- Percy documentation: https://docs.percy.io/
- Percy + Cypress integration guide: https://www.browserstack.com/docs/percy/cypress/getting-started
- Cypress documentation: https://docs.cypress.io/
- GitHub Actions docs: https://docs.github.com/actions