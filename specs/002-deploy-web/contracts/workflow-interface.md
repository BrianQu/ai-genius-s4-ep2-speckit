# Workflow Interface Contract: 002-deploy-web.yml

**Feature**: 002-deploy-web | **Date**: 2026-05-18  
**Artifact**: `.github/workflows/002-deploy-web.yml`

## Triggers

| Trigger | Condition | Default Environment |
|---|---|---|
| `push` | Branch = `main` | `dev` |
| `workflow_dispatch` | Manual — environment input required | As selected |

PR events do **not** trigger this workflow (FR-012 — explicitly out of scope).

## Inputs (workflow_dispatch only)

| Input | Type | Required | Default | Options |
|---|---|---|---|---|
| `environment` | choice | true | `dev` | `dev`, `qa`, `prod` |

## Concurrency

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

A newer run on the same workflow + ref cancels any in-flight run, preventing stale deployments (FR-008, SC-003).

## Required Secrets (per GitHub Environment)

| Secret | Source | Consumed By |
|---|---|---|
| `AZURE_CREDENTIALS` | Azure service principal JSON | `azure/login@v1` |
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | Azure SWA deployment token (per environment) | `Azure/static-web-apps-deploy@v1` |
| `GITHUB_TOKEN` | Auto-provided by GitHub Actions | `Azure/static-web-apps-deploy@v1` (`repo_token`) |

## Required Variables (per GitHub Environment)

| Variable | Consumed By | Example Value |
|---|---|---|
| `VITE_API_URL` | Build step `env:` | `https://aigenius4-api-dev.azurewebsites.net` |

## Required Variables (repository-level)

| Variable | Example Value |
|---|---|
| `APP_NAME` | `aigenius4` |

## Job: `deploy`

| Property | Value |
|---|---|
| Runner | `ubuntu-latest` |
| GitHub Environment | `${{ github.event.inputs.environment \|\| 'dev' }}` |

**Steps (in order)**:

| Step | Action / Command |
|---|---|
| Checkout code | `actions/checkout@v4` |
| Azure Login | `azure/login@v1` with `secrets.AZURE_CREDENTIALS` |
| Set up Node.js | `actions/setup-node@v4` — Node 20, npm cache keyed to `src/ai-genius-web/package-lock.json` |
| Install dependencies | `npm ci` in `src/ai-genius-web/` |
| Build frontend | `npm run build` in `src/ai-genius-web/` with `VITE_API_URL: ${{ vars.VITE_API_URL }}` |
| Deploy to SWA | `Azure/static-web-apps-deploy@v1` — `app_location: src/ai-genius-web/dist`, `output_location: ""`, `skip_app_build: true` |

## Side Effects

- Static assets in `src/ai-genius-web/dist/` are uploaded to the Azure Static Web App for the selected environment.
- No Azure infrastructure is modified or created (SWA resources are provisioned by `001-deploy-infra.yml`).
- No secrets or tokens are echoed to workflow logs.

## Failure Behaviour

| Step | On Failure |
|---|---|
| `npm ci` | Workflow fails; no build or deploy occurs |
| `npm run build` | Workflow fails; no deploy occurs; previous live content unchanged |
| `azure/login@v1` | Workflow fails; no deploy occurs |
| `Azure/static-web-apps-deploy@v1` | Workflow fails; previous live content remains unchanged |
