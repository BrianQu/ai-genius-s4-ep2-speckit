---
feature: 002-deploy-web
risk: low
breaking: false
reviewer-team: spec-reviewer
---

# Feature Specification: React Frontend CI/CD Deployment

**Feature Branch**: `002-deploy-web`  
**Created**: 2026-05-18  
**Status**: Draft  
**Input**: User description: "Deploy the AI Genius React frontend web app via GitHub Actions. The frontend is a React + Vite application in src/ai-genius-web."

## Clarifications

### Session 2026-05-18

- Q: How should `VITE_API_URL` be made available during the `npm run build` step? → A: Set `VITE_API_URL: ${{ vars.VITE_API_URL }}` as an env var on the build step (Vite standard).
- Q: Should pull request events trigger preview deployments on Azure Static Web Apps? → A: Out of scope — only `push` to `main` and `workflow_dispatch` trigger deployments.
- Q: What are the exact GitHub Environment names used for deployment protection rules? → A: `dev`, `qa`, `prod` (short-form, matching existing Bicep parameter files and AGENTS.md).
- Q: What is the expected maximum end-to-end deployment duration for SC-004? → A: 25 minutes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Automatic Deployment on Code Push (Priority: P1)

A developer merges a pull request to the main branch. Without any manual steps, the updated frontend is automatically built and deployed to the development environment on Azure Static Web Apps.

**Why this priority**: This is the core CI/CD value — every merge to main must produce a live, up-to-date deployment automatically. Removing manual steps reduces errors and speeds up delivery.

**Independent Test**: Push a change to `src/ai-genius-web/` on `main`. Observe that a workflow run starts, completes successfully, and the change appears on the dev Static Web App URL — with no other workflow involvement.

**Acceptance Scenarios**:

1. **Given** a developer pushes a commit to `main`, **When** the workflow triggers, **Then** the frontend dependencies are installed, the app is built, and the build artifact is deployed to the dev Static Web App within a reasonable time.
2. **Given** a push triggers a deployment, **When** a second push occurs before the first deployment finishes, **Then** the first deployment is cancelled and only the newer deployment proceeds.
3. **Given** the deployment completes, **When** a user visits the Static Web App URL, **Then** the latest version of the frontend is served.

---

### User Story 2 - Manual Environment-Targeted Deployment (Priority: P2)

A developer or release manager manually triggers a deployment and selects a specific target environment (dev, qa, or prod) from the workflow dispatch UI.

**Why this priority**: Teams need the ability to promote builds to higher environments on demand without pushing new code. This is the standard release promotion pattern.

**Independent Test**: Manually trigger the workflow via `workflow_dispatch`, select `qa` as the environment. Verify the deployment targets the qa Static Web App and not the dev or prod environment.

**Acceptance Scenarios**:

1. **Given** a user triggers the workflow manually, **When** they select `dev`, `qa`, or `prod`, **Then** the deployment targets the correct Static Web App environment.
2. **Given** no environment is explicitly provided (e.g., direct push), **When** the workflow determines the target, **Then** it defaults to the `dev` environment.
3. **Given** the manual deployment completes, **When** a user visits the selected environment's URL, **Then** the deployed frontend reflects the latest build.

---

### User Story 3 - Deployment Observability & Failure Handling (Priority: P3)

A developer monitors the GitHub Actions run for the frontend deployment. If the build or deployment fails, the failure is clearly reported in the workflow log so the developer can diagnose and fix the issue.

**Why this priority**: Visibility into deployment outcomes (pass/fail/in-progress) reduces time-to-resolution for broken deployments.

**Independent Test**: Introduce a syntax error in a frontend file, push to main, and verify the workflow fails with a clear error message — without silently deploying a broken build.

**Acceptance Scenarios**:

1. **Given** the frontend build succeeds, **When** the deployment step runs, **Then** the workflow reports a successful deployment status.
2. **Given** the frontend build fails (e.g., bad code), **When** the build step runs, **Then** the workflow stops immediately and reports a failure — no deployment occurs.
3. **Given** a deployment is in progress, **When** the GitHub Actions UI is viewed, **Then** the run status accurately reflects the current stage (building, deploying, succeeded, failed).

---

### Edge Cases

- What happens when `AZURE_CREDENTIALS` or `AZURE_STATIC_WEB_APPS_API_TOKEN` secrets are missing or invalid?
- What happens when `npm ci` fails due to a corrupted or missing `package-lock.json`?
- What happens when the build output (`dist/`) is empty or the build step is skipped?
- How does the workflow behave if two pushes arrive nearly simultaneously on `main`?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST automatically trigger a frontend deployment whenever a commit is pushed to the `main` branch.
- **FR-002**: The system MUST support a manual deployment trigger that allows the operator to select the target environment (dev, qa, or prod).
- **FR-003**: The system MUST default to the `dev` environment when no explicit environment is specified (e.g., on a push event).
- **FR-004**: The system MUST install all frontend dependencies in a clean, reproducible manner before building.
- **FR-005**: The system MUST produce a production-ready static build of the React 18 + Vite frontend, with environment-specific values embedded at build time.
- **FR-011**: The system MUST inject `VITE_API_URL` as a build-time environment variable — sourced from the per-environment GitHub Actions repository variable `vars.VITE_API_URL` — so the correct API endpoint is embedded in the static bundle.
- **FR-006**: The system MUST deploy the built static assets to the Azure Static Web Apps service using the deployment token stored in repository secrets.
- **FR-007**: The system MUST authenticate with Azure using the Azure credentials stored in repository secrets prior to deployment.
- **FR-008**: The system MUST cancel any in-progress deployment for the same workflow and branch when a newer deployment is triggered, to avoid deploying stale builds.
- **FR-009**: The system MUST follow the same environment selection and concurrency pattern established by the infrastructure deployment workflow (`001-deploy-infra.yml`).
- **FR-010**: The system MUST clearly report success or failure for each deployment in the workflow run log.
- **FR-012**: The workflow MUST NOT trigger on pull request events; PR preview environment deployments are explicitly out of scope.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Every push to `main` results in an automatic deployment attempt with a clear pass or fail outcome — zero silent failures.
- **SC-002**: Developers can deploy to any of the three environments (dev, qa, prod) without modifying any workflow file.
- **SC-003**: When two deployments are triggered in quick succession for the same branch, only the most recent one completes — stale deployments are cancelled.
- **SC-004**: A complete deployment cycle (from code push to live site) completes within 25 minutes.
- **SC-005**: The deployment workflow is consistent with the infrastructure workflow in structure and configuration, enabling teams to onboard with a single mental model.

## Assumptions

- The Azure Static Web App resources are already provisioned (by `001-deploy-infra.yml`) before this workflow runs.
- Repository secrets `AZURE_CREDENTIALS` and `AZURE_STATIC_WEB_APPS_API_TOKEN` are pre-configured in GitHub.
- The GitHub Environments are named exactly `dev`, `qa`, and `prod` in the repository settings and carry the appropriate protection rules per tier.
- `VITE_API_URL` is a per-environment GitHub Actions repository variable (`vars.VITE_API_URL`) injected as a build-time environment variable during `npm run build`.
- `ENVIRONMENT` and `APP_NAME` are GitHub environment-scoped variables available to the workflow.
- The frontend source is exclusively under `src/ai-genius-web/` and the build output lands in `src/ai-genius-web/dist/`.
- The frontend uses React 18 + Vite; Node.js 20.x is the required runtime for the build.
- Deployment to Azure Static Web Apps uses the `Azure/static-web-apps-deploy@v1` action pointed at the pre-built `dist/` folder.
- The environment-selection and concurrency pattern from `001-deploy-infra.yml` is the project standard and will be replicated here verbatim.
