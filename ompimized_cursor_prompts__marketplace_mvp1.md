# MVP 1 — 10-Day Cursor Prompts (TDD + Token Optimized) - use this [file open](https://github.com/Sukhrobjon/IlmAI.uz/blob/main/marketplace_ai_reference_card.md) as reference

## Prompt Rules
- One task per prompt
- Bullet points only
- No code examples
- Reference existing files
- Test after every 2-3 features

---

# DAY 1: Database & Auth

## 1.1 — User Model
```
Create SQLAlchemy model in /app/db/models/user.py

User table:
- id: UUID, primary key
- email: unique, indexed
- password_hash: string
- name: string
- subscription_tier: enum (starter/seller/business/agency), default starter
- workflow_runs_used: int, default 0
- workflow_runs_limit: int, default 30
- is_active: bool, default true
- created_at, updated_at: timestamps

Use SQLAlchemy 2.0 async syntax.
```

## 1.2 — Database Session
```
Create async database setup in /app/db/session.py

- AsyncSession factory
- get_db dependency for FastAPI
- Connection from DATABASE_URL env var
```

## 1.3 — Auth Schemas
```
Create Pydantic schemas in /app/schemas/auth.py

- RegisterInput: email, password, name
- LoginInput: email, password
- TokenResponse: access_token, refresh_token, token_type
- UserResponse: id, email, name, subscription_tier, workflow_runs_used, workflow_runs_limit
```

## 1.4 — Security Utils
```
Create /app/core/security.py

Functions:
- hash_password(password) -> str
- verify_password(plain, hashed) -> bool
- create_access_token(user_id, email) -> str (15 min expiry)
- create_refresh_token(user_id) -> str (7 day expiry)
- decode_token(token) -> dict

Use passlib bcrypt, python-jose.
Env vars: JWT_SECRET_KEY, JWT_ALGORITHM
```

## 1.5 — Auth Dependencies
```
Create /app/core/deps.py

- get_current_user: extracts JWT, returns User
- get_current_active_user: checks is_active

Raise 401 if invalid token.
```

## 1.6 — Auth Endpoints
```
Create /app/api/routes/auth.py

POST /register - create user, return tokens
POST /login - verify creds, return tokens
POST /refresh - refresh tokens
GET /me - return current user

Use schemas from /app/schemas/auth.py
Use security from /app/core/security.py
```

## 1.7 — TEST: Auth
```
Create /tests/test_auth.py

Tests:
- test_register_success
- test_register_duplicate_email
- test_login_success
- test_login_wrong_password
- test_get_me_authenticated
- test_get_me_no_token

Use pytest-asyncio, httpx AsyncClient.
Create test DB fixture in conftest.py.
```

## 1.8 — Fix failing tests
```
Review test output. Fix any failing tests in auth endpoints.
```

---

# DAY 2: Workflow Models & Engine

## 2.1 — Workflow Models
```
Create /app/db/models/workflow.py

Workflow table:
- id: UUID
- user_id: FK to users
- workflow_type: enum (niche_validation/competitor_analysis/opportunity_finder)
- status: enum (pending/queued/running/completed/failed)
- input_data: JSONB
- output_data: JSONB nullable
- error_message: text nullable
- started_at, completed_at: timestamps nullable
- created_at

WorkflowStep table:
- id: UUID
- workflow_id: FK
- step_name: string
- step_order: int
- status: enum
- input_data, output_data: JSONB
- api_provider: string
- tokens_used: int
- api_cost: decimal
- started_at, completed_at

Add indexes on user_id, status.
```

## 2.2 — Workflow Schemas
```
Create /app/schemas/workflow.py

Inputs:
- NicheValidationInput: niche (str), marketplace (literal), include_reviews (bool), max_competitors (int)
- CompetitorAnalysisInput: competitor_url (str), marketplace, include_price_history, include_reviews
- OpportunityFinderInput: budget (int min 50000), category (optional), risk_level (literal), experience_level (literal)

Responses:
- WorkflowCreate: id, workflow_type, status, created_at
- WorkflowStatus: id, status, current_step, progress_percent
- WorkflowDetail: all fields including steps
- WorkflowListItem: id, workflow_type, status, created_at, completed_at
```

## 2.3 — Workflow Base Classes
```
Create /app/workflows/base.py

Abstract classes:
- WorkflowStep: name, api_provider, execute(input, context), estimate_cost(input)
- Workflow: name, workflow_type, steps list, run(input, workflow_id)
- WorkflowContext: workflow_id, user_id, db, accumulated_data, total_cost

Include step progress tracking, error handling.
```

## 2.4 — Celery Setup
```
Create /app/tasks/celery_app.py

- Celery app with Redis broker
- Task time limit 600s
- JSON serialization

Create /app/tasks/workflow_tasks.py

- run_workflow_task(workflow_id, workflow_type, input_data)
- Updates workflow status in DB
- Executes workflow
- Handles errors, updates status to failed
```

## 2.5 — Workflow Endpoints
```
Create /app/api/routes/workflows.py

POST /research/niche-validation - create workflow, queue task
POST /research/competitor-analysis
POST /research/opportunity-finder
GET / - list user workflows, paginated
GET /{id} - workflow detail
GET /{id}/status - status only
DELETE /{id} - cancel if pending

Check workflow_runs_limit before creating.
Increment workflow_runs_used on create.
```

## 2.6 — TEST: Workflow CRUD
```
Create /tests/test_workflows.py

Tests:
- test_create_niche_validation
- test_create_invalid_input
- test_create_exceeds_limit
- test_list_workflows
- test_get_workflow_detail
- test_get_workflow_not_found
- test_delete_pending_workflow

Mock Celery task (don't actually run).
```

## 2.7 — Fix failing tests
```
Review test output. Fix any issues in workflow endpoints.
```

---

# DAY 3: API Integrations

## 3.1 — Base API Client
```
Create /app/integrations/base.py

APIClient base class:
- Async HTTP with httpx
- Retry logic (3 attempts, exponential backoff)
- Request logging
- Custom exceptions: APIError, RateLimitError
```

## 3.2 — Anthropic Client
```
Create /app/integrations/anthropic_client.py

AnthropicClient:
- generate(prompt, system, model, max_tokens) -> (text, usage)
- generate_json(prompt, system, model) -> dict (parse JSON response)
- estimate_cost(input_tokens, output_tokens, model) -> float

Models: claude-3-5-haiku-20241022, claude-3-5-sonnet-20241022
Env: ANTHROPIC_API_KEY
```

## 3.3 — OpenAI Client
```
Create /app/integrations/openai_client.py

OpenAIClient:
- analyze_image(image_url, prompt, model) -> (text, usage)
- analyze_images_batch(urls, prompt) -> list

Model: gpt-4o-mini
Env: OPENAI_API_KEY
Parse JSON from response.
```

## 3.4 — Apify Client
```
Create /app/integrations/apify_client.py

ApifyClient:
- run_actor(actor_id, input_data, timeout) -> list[dict]
- scrape_wildberries_search(query, max_results) -> list
- scrape_product_reviews(url, max_reviews) -> list

Env: APIFY_API_TOKEN
Normalize output data structure.
```

## 3.5 — Cache Service
```
Create /app/services/cache_service.py

CacheService:
- get(cache_key) -> dict or None
- set(cache_key, data, ttl_hours)
- get_or_fetch(cache_key, fetch_func, ttl)
- generate_cache_key(prefix, **params) -> str

Use research_cache table.
```

## 3.6 — TEST: Integrations
```
Create /tests/test_integrations.py

Tests (mock external APIs):
- test_anthropic_generate
- test_anthropic_generate_json
- test_openai_analyze_image
- test_apify_run_actor
- test_cache_get_set
- test_cache_get_or_fetch

Use pytest-mock to mock HTTP calls.
```

## 3.7 — Fix failing tests
```
Fix any integration issues from test results.
```

---

# DAY 4: Niche Validation Workflow

## 4.1 — Scrape Steps
```
Create /app/workflows/research/steps/scrape_steps.py

ScrapeMarketDataStep:
- Uses ApifyClient
- Scrapes top 50 products
- Returns: products list, price_stats

ScrapeReviewsStep:
- Scrapes reviews for top 5 products
- Returns: reviews list
```

## 4.2 — Analysis Steps
```
Create /app/workflows/research/steps/analysis_steps.py

AnalyzePhotosStep:
- Uses OpenAI Vision
- Analyzes top 10 product images
- Returns: photo_analyses list

AnalyzeSentimentStep:
- Uses Claude Haiku
- Analyzes all reviews
- Returns: sentiment_analysis dict
```

## 4.3 — Synthesis Step
```
Create /app/workflows/research/steps/synthesis_steps.py

SynthesizeNicheReportStep:
- Uses Claude Sonnet
- Takes all previous data
- Returns: full report with go/no-go recommendation
```

## 4.4 — Niche Validation Prompts
```
Create /app/workflows/research/prompts/niche_validation.py

Constants:
- PHOTO_ANALYSIS_PROMPT: analyze product photo, return JSON
- SENTIMENT_SYSTEM_PROMPT: role definition
- SENTIMENT_USER_PROMPT: template with {reviews_text}, request JSON
- SYNTHESIS_SYSTEM_PROMPT: marketplace strategist role
- SYNTHESIS_USER_PROMPT: template with all data, request full report JSON
```

## 4.5 — Niche Workflow Class
```
Create /app/workflows/research/niche_validation.py

NicheValidationWorkflow:
- Chains all 5 steps
- Register in workflow registry

Create /app/workflows/registry.py if not exists.
```

## 4.6 — TEST: Niche Workflow
```
Create /tests/test_niche_validation.py

Tests (mock all APIs):
- test_scrape_market_data_step
- test_analyze_photos_step
- test_analyze_sentiment_step
- test_synthesize_report_step
- test_full_workflow_execution

Use fixtures for mock API responses.
```

## 4.7 — Fix + Integration Test
```
Fix failing tests.

Add integration test:
- test_niche_validation_api_to_completion
- Create workflow via API
- Mock Celery to run sync
- Verify final output structure
```

---

# DAY 5: Other Workflows

## 5.1 — Competitor Steps
```
Create /app/workflows/research/steps/competitor_steps.py

ExtractProductDataStep: scrape single product
GetPriceHistoryStep: get 90-day prices
AnalyzeCompetitorReviewsStep: sentiment on competitor reviews
GenerateVulnerabilityReportStep: Claude Sonnet synthesis
```

## 5.2 — Competitor Prompts
```
Create /app/workflows/research/prompts/competitor_analysis.py

- COMPETITOR_REVIEW_PROMPT
- VULNERABILITY_REPORT_PROMPT
```

## 5.3 — Competitor Workflow
```
Create /app/workflows/research/competitor_analysis.py

CompetitorAnalysisWorkflow with all steps.
Register in registry.
```

## 5.4 — Opportunity Steps
```
Create /app/workflows/research/steps/opportunity_steps.py

IdentifyCategoriesStep: suggest categories based on budget
ScanCategoriesStep: scrape multiple categories
CalculateOpportunityScoresStep: score and rank
DeepDiveTopOpportunitiesStep: analyze top 3
GenerateOpportunityReportStep: final report
```

## 5.5 — Opportunity Prompts + Workflow
```
Create /app/workflows/research/prompts/opportunity_finder.py
Create /app/workflows/research/opportunity_finder.py

Register in registry.
```

## 5.6 — TEST: All Workflows
```
Create /tests/test_competitor_analysis.py
Create /tests/test_opportunity_finder.py

Test each step + full workflow (mocked).
```

## 5.7 — Fix + Verify All Workflows
```
Fix failing tests.
Run all workflow tests together.
Verify registry has all 3 workflows.
```

---

# DAY 6: Frontend Setup

## 6.1 — Next.js Project
```
Initialize Next.js 14 in /frontend

- App Router
- TypeScript
- Tailwind CSS
- Dark theme default

Create folder structure:
/app/(auth), /(dashboard)
/components/ui, /auth, /dashboard, /workflows
/lib, /hooks, /stores, /types
```

## 6.2 — shadcn/ui Setup
```
Install shadcn/ui components:
- Button, Input, Card, Dialog
- Toast, Dropdown, Table
- Badge, Progress, Tabs

Configure dark theme.
```

## 6.3 — API Client + Types
```
Create /lib/api.ts

- Axios instance with baseURL from env
- Request interceptor: add JWT
- Response interceptor: handle 401

Create /types/index.ts

- User, Workflow, WorkflowStep types
- API response types
```

## 6.4 — Auth Store
```
Create /stores/auth-store.ts

Zustand store:
- user, accessToken, isAuthenticated
- login(email, password)
- register(email, password, name)
- logout()
- loadFromStorage()

Persist tokens to localStorage.
```

## 6.5 — Auth Pages
```
Create /app/(auth)/layout.tsx - centered auth layout
Create /app/(auth)/login/page.tsx
Create /app/(auth)/register/page.tsx

Use react-hook-form + zod validation.
Connect to auth store.
Redirect to /dashboard on success.
```

## 6.6 — TEST: Auth Flow
```
Manual testing checklist:
- [ ] Register new user
- [ ] Login with credentials
- [ ] Token stored in localStorage
- [ ] Redirect to dashboard
- [ ] Logout clears token

Fix any issues found.
```

---

# DAY 7: Dashboard & Workflow List

## 7.1 — Dashboard Layout
```
Create /app/(dashboard)/layout.tsx

- Protected route (redirect if no token)
- Sidebar: logo, nav items, usage indicator
- Header: page title, user menu
- Main content area

Create /components/dashboard/sidebar.tsx
Create /components/dashboard/header.tsx
```

## 7.2 — Workflow Store + Hook
```
Create /stores/workflow-store.ts

- workflows list
- filters (status, type)
- pagination

Create /hooks/use-workflows.ts

- Fetch workflows from API
- Refetch on filter change
- Loading/error states
```

## 7.3 — Workflows List Page
```
Create /app/(dashboard)/workflows/page.tsx

- Page title, "New Workflow" button
- Filters row
- Workflow cards grid
- Empty state
- Pagination

Create /components/workflows/workflow-card.tsx
- Type icon, name, status badge, date
- Click to view detail
```

## 7.4 — New Workflow Page
```
Create /app/(dashboard)/workflows/new/page.tsx

- Workflow type selector (3 cards)
- Click navigates to /workflows/new/[type]

Create /components/workflows/workflow-type-selector.tsx
```

## 7.5 — Workflow Forms
```
Create /components/workflows/forms/niche-form.tsx
Create /components/workflows/forms/competitor-form.tsx
Create /components/workflows/forms/opportunity-form.tsx

Each form:
- Fields per schema
- Zod validation
- Submit calls API
- Shows success modal
```

## 7.6 — TEST: Workflow Creation
```
Manual testing:
- [ ] List page shows workflows
- [ ] Filters work
- [ ] Create each workflow type
- [ ] Form validation works
- [ ] Success modal appears
- [ ] New workflow in list

Fix issues.
```

---

# DAY 8: Workflow Detail & Results

## 8.1 — Workflow Detail Page
```
Create /app/(dashboard)/workflows/[id]/page.tsx

- Fetch workflow by ID
- Show progress view if running
- Show results if completed
- Show error if failed
- Poll for status updates

Create /hooks/use-workflow-detail.ts
```

## 8.2 — Progress Component
```
Create /components/workflows/workflow-progress.tsx

- Step list with status icons
- Current step highlighted
- Progress bar
- Auto-refresh every 3s
```

## 8.3 — Niche Results Component
```
Create /components/workflows/results/niche-results.tsx

Sections:
- Executive summary + GO/NO-GO badge
- Market overview stats
- Competition analysis
- Customer insights
- Action plan

Use Cards, Tables, Badges.
```

## 8.4 — Competitor Results Component
```
Create /components/workflows/results/competitor-results.tsx

Sections:
- Competitor overview
- Strengths/weaknesses
- Pricing analysis
- How to beat them
```

## 8.5 — Opportunity Results Component
```
Create /components/workflows/results/opportunity-results.tsx

Sections:
- Top opportunities ranked
- Expandable details
- Financial projections
- Recommended action
```

## 8.6 — TEST: Results Display
```
Manual testing with mock data:
- [ ] Progress shows correctly
- [ ] Each result type renders
- [ ] All sections visible
- [ ] Mobile responsive

Fix styling issues.
```

---

# DAY 9: Polish & Settings

## 9.1 — Error Handling
```
Create /components/common/api-error.tsx

- Different messages per status code
- Retry button for 500
- Login redirect for 401

Add error boundaries to layouts.
Add toast notifications for errors.
```

## 9.2 — Loading States
```
Add skeleton loaders to:
- Workflow list
- Workflow detail
- Results sections

Create /components/ui/skeleton.tsx if needed.
```

## 9.3 — Usage Tracking UI
```
Create /components/dashboard/usage-widget.tsx

- Shows X / Y workflows used
- Progress bar
- Warning at 80%
- Block at 100%

Add to sidebar.
```

## 9.4 — Settings Page
```
Create /app/(dashboard)/settings/page.tsx

Tabs:
- Profile: name update
- Security: change password
- Subscription: current plan, usage

Create form components for each tab.
```

## 9.5 — TEST: Full User Flow
```
End-to-end manual test:
- [ ] Register
- [ ] Login
- [ ] Create workflow
- [ ] View progress
- [ ] See results
- [ ] Check usage updated
- [ ] Update settings
- [ ] Logout

Document any bugs.
```

## 9.6 — Fix All Bugs
```
Fix all issues from E2E testing.
```

---

# DAY 10: Testing & Deploy

## 10.1 — Backend Test Coverage
```
Run pytest --cov

Add missing tests to reach 80% on:
- /app/api/routes/
- /app/services/
- /app/workflows/

Focus on critical paths.
```

## 10.2 — Frontend Tests
```
Add Vitest tests for:
- Auth store actions
- Workflow hooks
- Form validation

Create /tests/setup.ts with MSW handlers.
```

## 10.3 — Docker Production
```
Create /docker-compose.prod.yml

- Production settings
- Health checks
- Restart policies
- Resource limits

Create /backend/Dockerfile.prod (multi-stage)
Create /frontend/Dockerfile.prod (build + nginx)
```

## 10.4 — Documentation
```
Update README.md:
- Setup instructions
- Environment variables
- Running locally
- Running tests
- Deployment

Create /docs/api.md with endpoint reference.
```

## 10.5 — Deploy Checklist
```
Verify:
- [ ] All tests pass
- [ ] Docker builds work
- [ ] Env vars documented
- [ ] DB migrations run
- [ ] Health checks pass
- [ ] CORS configured
- [ ] Secrets not in code

Create deployment script.
```

## 10.6 — Final Review
```
Code review checklist:
- [ ] No console.logs in production
- [ ] Error handling everywhere
- [ ] Types complete
- [ ] No hardcoded values
- [ ] Comments on complex logic

Tag release v0.1.0
```

---

# DAILY RHYTHM

```
Morning (2-3 prompts):
  Build features

Midday (1 prompt):
  Write tests for morning work

Afternoon (2-3 prompts):
  Build more features

End of day (1 prompt):
  Test + fix failures
```

---

# PROMPT TIPS

1. **Copy prompt exactly** — Don't add extra context
2. **Let Cursor read your files** — It knows your codebase
3. **Fix immediately** — Don't accumulate bugs
4. **Small commits** — After each working feature
5. **Run tests often** — `pytest` / `npm test`

---

# FILE REFERENCE

After Day 5, backend structure:
```
/app
  /api/routes/auth.py, workflows.py
  /core/config.py, security.py, deps.py
  /db/models/user.py, workflow.py
  /db/session.py, base.py
  /schemas/auth.py, workflow.py
  /services/cache_service.py
  /integrations/base.py, anthropic_client.py, openai_client.py, apify_client.py
  /workflows/base.py, registry.py
    /research/niche_validation.py, competitor_analysis.py, opportunity_finder.py
      /steps/scrape_steps.py, analysis_steps.py, synthesis_steps.py
      /prompts/niche_validation.py, competitor_analysis.py, opportunity_finder.py
  /tasks/celery_app.py, workflow_tasks.py
```

After Day 9, frontend structure:
```
/app
  /(auth)/login, register
  /(dashboard)/page, workflows, workflows/[id], workflows/new, settings
/components
  /ui/button, input, card, etc
  /dashboard/sidebar, header, usage-widget
  /workflows/workflow-card, workflow-progress, forms/, results/
  /common/api-error
/lib/api.ts
/stores/auth-store.ts, workflow-store.ts
/hooks/use-workflows.ts, use-workflow-detail.ts
/types/index.ts
```
