# Data Model: React Frontend CI/CD Deployment

**Phase 1 — Environment & Configuration Schema**  
**Feature**: 002-deploy-web | **Date**: 2026-05-18

## Entities

### Workflow Run

Represents a single execution of `002-deploy-web.yml`.

| Field | Type | Description |
|---|---|---|
| `trigger` | enum | `push` or `workflow_dispatch` |
| `environment` | enum | `dev`, `qa`, or `prod` |
| `ref` | string | Git ref that initiated the run (e.g., `refs/heads/main`) |
| `status` | enum | `queued`, `in_progress`, `success`, `failure`, `cancelled` |

**State Transitions**:
- `push` to `main` → environment resolves to `dev` (default)
- `workflow_dispatch` with explicit environment input → environment = selected value
- Newer run triggered while a run for the same workflow + ref is `in_progress` → previous run transitions to `cancelled`

---

### GitHub Environment Configuration

Each of the three GitHub Environments carries the following scoped values:

| Environment | Name | Kind | Description |
|---|---|---|---|
| `dev` / `qa` / `prod` | `AZURE_STATIC_WEB_APPS_API_TOKEN` | Secret | Deployment token for the SWA instance in this environment tier |
| `dev` / `qa` / `prod` | `AZURE_CREDENTIALS` | Secret | Azure service principal JSON consumed by `azure/login@v1` |
| `dev` / `qa` / `prod` | `VITE_API_URL` | Variable | API base URL embedded in the static bundle at build time |
| `dev` / `qa` / `prod` | `AZURE_RESOURCE_GROUP` | Variable | Azure resource group for this environment (reference; used by infra workflow) |

**Repository-level variables** (not environment-scoped):

| Name | Kind | Description |
|---|---|---|
| `APP_NAME` | Variable | Application base name (e.g., `aigenius4`) |

---

### Build Artifact

The static bundle produced by `npm run build` for a single workflow run.

| Field | Description |
|---|---|
| Source directory | `src/ai-genius-web/` |
| Output directory | `src/ai-genius-web/dist/` |
| Produced by | `npm run build` (Vite bundler) |
| Committed to repo | No — generated at CI time only |
| Deployment action | `Azure/static-web-apps-deploy@v1` with `skip_app_build: true` |
| Baked-in values | `VITE_API_URL` from the active environment's `vars.VITE_API_URL` |

**Validation rules**:
- `dist/` must be non-empty after `npm run build`; the deploy step assumes a populated directory.
- If `npm run build` fails, the workflow fails immediately and the deploy step is skipped.
