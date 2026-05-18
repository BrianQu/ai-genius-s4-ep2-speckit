# Quickstart: 002-deploy-web Workflow

**Feature**: 002-deploy-web | **Date**: 2026-05-18

## Prerequisites

Before the workflow can run successfully, the following must be in place:

1. **Azure infrastructure provisioned**: Run `001-deploy-infra.yml` first to ensure the Azure Static Web Apps exist for `dev`, `qa`, and `prod`.

2. **GitHub Environments configured**: Three environments — `dev`, `qa`, `prod` — must exist under repository **Settings → Environments**.

3. **Secrets per environment**: In each GitHub Environment, add:
   - `AZURE_CREDENTIALS` — Azure service principal JSON (`clientId`, `clientSecret`, `subscriptionId`, `tenantId`)
   - `AZURE_STATIC_WEB_APPS_API_TOKEN` — deployment token from the corresponding Azure SWA resource (see below)

4. **Variables per environment**: In each GitHub Environment, add:
   - `VITE_API_URL` — API base URL for that tier (e.g., `https://aigenius4-api-dev.azurewebsites.net`)

5. **Repository-level variable**: Under **Settings → Variables → Actions**, add:
   - `APP_NAME` — base application name (e.g., `aigenius4`)

## Getting the SWA Deployment Token

```bash
az staticwebapp secrets list \
  --name <static-web-app-name> \
  --resource-group <resource-group> \
  --query "properties.apiKey" \
  --output tsv
```

Or: Azure portal → Static Web App resource → **Settings → Deployment token**.

Repeat for each environment's SWA instance and store each token in the corresponding GitHub Environment as `AZURE_STATIC_WEB_APPS_API_TOKEN`.

---

## Test: Automatic Deployment (User Story 1 — P1)

1. Make a visible change to the frontend (e.g., edit `src/ai-genius-web/src/App.jsx`).
2. Commit and push to `main`.
3. Go to **GitHub → Actions → 002 Deploy Web to Azure Static Web Apps**.
4. Confirm a new run starts automatically.
5. Wait for the run to complete (expected: ≤25 min per SC-004).
6. Visit the `dev` Static Web App URL and confirm the change is live.

## Test: Manual Environment Deployment (User Story 2 — P2)

1. Go to **GitHub → Actions → 002 Deploy Web to Azure Static Web Apps** → **Run workflow**.
2. Select `qa` from the environment dropdown.
3. Click **Run workflow**.
4. Confirm the job runs with the `qa` GitHub Environment (visible in the run summary sidebar).
5. Visit the `qa` Static Web App URL and confirm the deployment.

## Test: Concurrency Cancellation (SC-003)

1. Trigger two pushes in quick succession to `main` (e.g., two fast commits).
2. Go to **GitHub → Actions**.
3. Confirm the first run shows **Cancelled** and only the second run proceeds to completion.

## Test: Build Failure Blocks Deploy (User Story 3 — P3)

1. Introduce a JavaScript syntax error in `src/ai-genius-web/src/App.jsx`.
2. Push to `main`.
3. Confirm the workflow fails at the **Build frontend** step.
4. Confirm the **Deploy to Azure Static Web Apps** step never runs.
5. Revert the error and push again to confirm recovery.
