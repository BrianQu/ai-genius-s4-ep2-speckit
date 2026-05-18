# Research: React Frontend CI/CD Deployment

**Phase 0 — Resolved Unknowns**  
**Feature**: 002-deploy-web | **Date**: 2026-05-18

## Decision Log

### 1. Azure Static Web Apps — Pre-Built Deployment Pattern

**Question**: How does `Azure/static-web-apps-deploy@v1` work when the build runs in GitHub Actions rather than on the SWA platform?

**Decision**: Use `skip_app_build: true` with `app_location` pointing to the pre-built `dist/` folder and `output_location` set to `""`.

**Rationale**: The SWA platform can build apps internally, but running the build in GitHub Actions gives full control over the Node.js version, environment variable injection (`VITE_API_URL`), and build logs. When `skip_app_build: true`, the action treats `app_location` as a finished static bundle and uploads it directly without attempting a second build. Setting `output_location: ""` signals that the `app_location` directory is itself the deployable output.

**Configuration**:
```yaml
- uses: Azure/static-web-apps-deploy@v1
  with:
    action: upload
    app_location: src/ai-genius-web/dist
    output_location: ""
    skip_app_build: true
    repo_token: ${{ secrets.GITHUB_TOKEN }}
    azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
```

**Alternatives considered**:
- Let SWA platform build the app — rejected because `VITE_API_URL` must be injected at build time and the SWA internal build environment cannot access GitHub Actions `vars.*` variables.

---

### 2. Per-Environment Deployment Tokens

**Question**: How is the correct `AZURE_STATIC_WEB_APPS_API_TOKEN` selected per environment when the same secret name is used across all three GitHub Environments?

**Decision**: Store `AZURE_STATIC_WEB_APPS_API_TOKEN` as a secret within each of the three GitHub Environments (`dev`, `qa`, `prod`) with the deployment token for the corresponding Azure SWA resource.

**Rationale**: When a workflow job declares `environment: dev`, GitHub Actions automatically resolves `secrets.AZURE_STATIC_WEB_APPS_API_TOKEN` to the value stored in the `dev` environment scope. No secret-name variation or matrix strategy is needed — the same workflow handles all three environments transparently.

**How to obtain each token**:
```bash
az staticwebapp secrets list \
  --name <static-web-app-name> \
  --resource-group <resource-group> \
  --query "properties.apiKey" \
  --output tsv
```

**Alternatives considered**:
- Distinct secret names per environment (e.g., `AZURE_SWA_TOKEN_DEV`, `AZURE_SWA_TOKEN_QA`) — rejected because it requires hardcoding environment-name suffixes in the workflow and diverges from the single-name pattern already used for `AZURE_CREDENTIALS`.

---

### 3. VITE_API_URL Injection at Build Time

**Question**: At what point in the workflow must `VITE_API_URL` be available, and how should it be passed?

**Decision**: Set as an `env:` key on the `npm run build` step only, sourced from `vars.VITE_API_URL` (GitHub Actions repository variable scoped to the active environment).

**Rationale**: Vite reads `VITE_*` environment variables from the Node.js process environment at build time and statically embeds them in the compiled bundle. The variable is only needed during the build step — it is not required at install time (`npm ci`) or deploy time. Scoping it to the build step keeps the workflow minimal and avoids unnecessary env-var proliferation.

**Configuration**:
```yaml
- name: Build frontend
  working-directory: src/ai-genius-web
  run: npm run build
  env:
    VITE_API_URL: ${{ vars.VITE_API_URL }}
```

**Alternatives considered**:
- Write a `.env` file before building — rejected for added complexity with no benefit; Vite's built-in `VITE_*` env-var mechanism is simpler and the established convention.
- Set `VITE_API_URL` as a top-level `env:` on the job — works, but unnecessarily exposes the value to all steps including deploy; step-level scoping is cleaner.
