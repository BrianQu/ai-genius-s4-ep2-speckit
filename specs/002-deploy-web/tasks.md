# Tasks: React Frontend CI/CD Deployment

**Feature**: `002-deploy-web`
**Generated**: 2026-05-18
**Input**: `specs/002-deploy-web/plan.md`, `spec.md`, `data-model.md`, `contracts/workflow-interface.md`, `research.md`, `quickstart.md`

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different sections/files, no shared dependencies)
- **[Story]**: Maps to user story in spec.md — US1, US2, US3
- Exact file paths are included in every task description

---

## Phase 1: Setup

**Purpose**: Create the workflow file scaffold with the correct workflow name

- [X] T001 Create `.github/workflows/002-deploy-web.yml` with `name: 002 Deploy Web to Azure Static Web Apps`

---

## Phase 2: Foundational — Concurrency & Environment Resolution

**Purpose**: Add the concurrency block and workflow-level env vars that ALL user stories depend on

**⚠️ CRITICAL**: No user story job runs correctly without the concurrency group and ENVIRONMENT resolution in place

- [X] T002 Add `concurrency` block (`group: ${{ github.workflow }}-${{ github.ref }}`, `cancel-in-progress: true`) and workflow-level `env` block (`ENVIRONMENT: ${{ github.event.inputs.environment || 'dev' }}`, `APP_NAME: ${{ vars.APP_NAME }}`) to `.github/workflows/002-deploy-web.yml`

**Checkpoint**: Concurrency and environment resolution in place — user story phases can now begin

---

## Phase 3: User Story 1 — Automatic Deployment on Code Push (Priority: P1) 🎯 MVP

**Goal**: Every push to `main` automatically installs Node.js 20 dependencies, builds the React 18 + Vite frontend with `VITE_API_URL` injected, and deploys the pre-built `dist/` to the dev Azure Static Web App.

**Independent Test**: Push a visible change to `src/ai-genius-web/src/App.jsx` on `main` → confirm workflow run starts automatically, completes, and the change is live on the dev SWA URL — with no manual steps.

- [X] T003 [US1] Add `on: push: branches: [main]` trigger to the `on:` block in `.github/workflows/002-deploy-web.yml` — write `on:` as a multi-key dict (not an inline shorthand) so that T010's `workflow_dispatch:` key can be added in Phase 4 without restructuring
- [X] T004 [US1] Add `deploy` job with `name: Build & Deploy Frontend`, `runs-on: ubuntu-latest`, and `environment: ${{ github.event.inputs.environment || 'dev' }}` under `jobs:` in `.github/workflows/002-deploy-web.yml`
- [X] T005 [US1] Add `Checkout code` step (`uses: actions/checkout@v4`) and `Azure Login` step (`uses: azure/login@v1`, `with: creds: ${{ secrets.AZURE_CREDENTIALS }}`) to the `deploy` job in `.github/workflows/002-deploy-web.yml`
- [X] T006 [US1] Add `Set up Node.js` step (`uses: actions/setup-node@v4`, `with: node-version: '20'`, `cache: 'npm'`, `cache-dependency-path: src/ai-genius-web/package-lock.json`) to the `deploy` job in `.github/workflows/002-deploy-web.yml`
- [X] T007 [US1] Add `Install dependencies` step (`working-directory: src/ai-genius-web`, `run: npm ci`) to the `deploy` job in `.github/workflows/002-deploy-web.yml`
- [X] T008 [US1] Add `Build frontend` step (`working-directory: src/ai-genius-web`, `run: npm run build`, `env: VITE_API_URL: ${{ vars.VITE_API_URL }}`) to the `deploy` job in `.github/workflows/002-deploy-web.yml`
- [X] T009 [US1] Add `Deploy to Azure Static Web Apps` step (`uses: Azure/static-web-apps-deploy@v1`, `with: action: upload`, `app_location: src/ai-genius-web/dist`, `output_location: ""`, `skip_app_build: true`, `repo_token: ${{ secrets.GITHUB_TOKEN }}`, `azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}`) to the `deploy` job in `.github/workflows/002-deploy-web.yml`

**Checkpoint**: US1 complete — pushing to `main` triggers a full build-and-deploy cycle to the `dev` SWA. All 6 steps are in place and run in order.

---

## Phase 4: User Story 2 — Manual Environment-Targeted Deployment (Priority: P2)

**Goal**: Operators can manually trigger a deployment to any of `dev`, `qa`, or `prod` from the GitHub Actions UI without pushing a new commit.

**Independent Test**: Navigate to Actions → 002 Deploy Web to Azure Static Web Apps → Run workflow, select `qa`, click Run workflow — confirm the job runs under the `qa` GitHub Environment and the qa SWA receives the new deployment.

- [X] T010 [P] [US2] Add `workflow_dispatch` trigger with `inputs.environment` (`type: choice`, `required: true`, `default: dev`, `options: [dev, qa, prod]`) to the `on:` block in `.github/workflows/002-deploy-web.yml`

**Checkpoint**: US2 complete — the workflow appears in the GitHub Actions "Run workflow" dropdown with an environment selector. Push and dispatch triggers share the same `ENVIRONMENT` resolution expression defined in T002.

---

## Phase 5: User Story 3 — Deployment Observability & Failure Handling (Priority: P3)

**Goal**: Each step has a descriptive `name:` label for clear GitHub Actions UI progress reporting, and the build step precedes the deploy step so a build failure blocks the upload.

**Independent Test**: Introduce a syntax error in `src/ai-genius-web/src/App.jsx`, push to `main` — confirm the workflow fails at the `Build frontend` step and the `Deploy to Azure Static Web Apps` step never executes.

> US3 has no standalone implementation task — its concerns are structurally enforced by the sequential step ordering established in T005–T009 (`Build frontend` precedes `Deploy to Azure Static Web Apps` by task order). Step name labels are specified inline within each step task. Verification is covered by T011 (Final Phase).

**Checkpoint**: US3 is satisfied when T005–T009 are complete and T011 validation passes.

---

## Final Phase: Polish & Cross-Cutting Concerns

- [X] T011 Validate the complete `.github/workflows/002-deploy-web.yml` — confirm: (1) YAML is syntactically valid; (2) `name:` / `on:` / `concurrency:` / `env:` / `jobs:` sections appear in the same top-level order as `.github/workflows/001-deploy-infra.yml`; (3) all `secrets.*` and `vars.*` references match the names in `specs/002-deploy-web/contracts/workflow-interface.md`; (4) the environment-selector expression `${{ github.event.inputs.environment || 'dev' }}` appears on both the `env.ENVIRONMENT` line and `jobs.deploy.environment`; (5) all 6 steps carry descriptive `name:` labels and `Build frontend` precedes `Deploy to Azure Static Web Apps` in step order; (6) no `pull_request:` key is present in the `on:` block (FR-012)

---

## Dependencies

```
Phase 1 (T001 — create file)
    │
    ▼
Phase 2 (T002 — concurrency + env)
    │
    ▼
Phase 3 / US1 (T003 → T004 → T005 → T006 → T007 → T008 → T009)
    │
    ├──► Phase 4 / US2 (T010 — adds workflow_dispatch to on: block)  [P: separate YAML section]
    │
    └──► Phase 5 / US3 (no task — enforced by T005–T009 step ordering)
              │
    ──────────┘
    │
    ▼
Final Phase (T011 — full structural + FR-012 validation)
```

**User story completion order**:
1. **US1** (P1) — must complete first; creates the workflow file all other stories extend
2. **US2** (P2) — can run in parallel with US3 after US1 (modifies `on:` block; different YAML section from `jobs:`)
3. **US3** (P3) — enforced structurally by T005–T009 step ordering; no separate implementation task

---

## Parallel Execution Examples

### After US1 — US2 and US3 run in parallel

```
After T009 (US1 complete):
  Agent A ──► T010  (US2 — add workflow_dispatch to on: block in 002-deploy-web.yml)
  → T011 (Final Phase — full structural + FR-012 validation, after T010 complete)
```

US3 has no parallel agent task — its concerns (step naming, build-before-deploy ordering) are enforced by T005–T009 execution order.

### Phase 3 — US1 steps are strictly sequential (single YAML file)

T003–T009 each extend the same `jobs.deploy` section; each step depends on the job definition established by the previous task.

---

## Implementation Strategy

**Suggested MVP scope — Phases 1–3 (US1 only, T001–T009)**:

Phases 1 + 2 + 3 deliver the complete automated push-to-dev pipeline — a fully working, independently testable system. US2 and US3 extend it but are not required for the first live deployment.

**Delivery order**:
1. T001–T009 (US1 MVP) → push to `main`, verify the dev SWA receives the deployment
2. T010 (US2) → add `workflow_dispatch` trigger; test manual `qa` deploy
3. T011 (Polish) → full structural validation including step names, FR-012 absence, and 001 pattern match
