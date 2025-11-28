# MVP 1 Quick Reference Card
## Keep This Open While Building

---

## PROJECT STRUCTURE

```
/backend
  /app
    /api/routes/       # FastAPI endpoints
    /core/             # Config, security, deps
    /db/models/        # SQLAlchemy models
    /schemas/          # Pydantic schemas
    /services/         # Business logic
    /workflows/        # Workflow engine
      /research/       # Research workflows
        /prompts/      # AI prompts
    /integrations/     # API clients
    /tasks/            # Celery tasks
  /tests/
  /alembic/
  main.py

/frontend
  /app/               # Next.js App Router
    /(auth)/          # Auth pages
    /(dashboard)/     # Protected pages
  /components/
    /ui/              # shadcn components
    /auth/
    /dashboard/
    /workflows/
  /lib/               # Utilities
  /hooks/             # Custom hooks
  /stores/            # Zustand stores
  /types/
```

---

## KEY COMMANDS

```bash
# Backend
cd backend
poetry install                    # Install deps
poetry run alembic upgrade head   # Run migrations
poetry run uvicorn app.main:app --reload  # Start API
poetry run celery -A app.tasks.celery_app worker  # Start worker
poetry run pytest                 # Run tests

# Frontend
cd frontend
npm install                       # Install deps
npm run dev                       # Start dev server
npm run build                     # Build production
npm run test                      # Run tests

# Docker
docker-compose up -d              # Start all services
docker-compose logs -f api        # View API logs
docker-compose down               # Stop all
```

---

## DATABASE MODELS

```
users
├── id (UUID, PK)
├── email (unique)
├── password_hash
├── name
├── subscription_tier (starter|seller|business|agency)
├── workflow_runs_used (int)
├── workflow_runs_limit (int)
└── created_at, updated_at

workflows
├── id (UUID, PK)
├── user_id (FK)
├── workflow_type (niche_validation|competitor_analysis|opportunity_finder)
├── status (pending|queued|running|completed|failed)
├── input_data (JSONB)
├── output_data (JSONB)
├── error_message
└── created_at, started_at, completed_at

workflow_steps
├── id (UUID, PK)
├── workflow_id (FK)
├── step_name
├── step_order
├── status
├── input_data, output_data (JSONB)
├── api_provider
├── tokens_used, api_cost
└── started_at, completed_at
```

---

## API ENDPOINTS

```
# Auth
POST   /api/auth/register     # Create account
POST   /api/auth/login        # Get tokens
POST   /api/auth/refresh      # Refresh token
GET    /api/auth/me           # Current user
PUT    /api/auth/me           # Update profile

# Workflows
POST   /api/workflows/research/niche-validation
POST   /api/workflows/research/competitor-analysis
POST   /api/workflows/research/opportunity-finder
GET    /api/workflows         # List (paginated)
GET    /api/workflows/{id}    # Detail
GET    /api/workflows/{id}/status  # Status only
DELETE /api/workflows/{id}    # Cancel/delete

# Usage
GET    /api/usage/summary     # Current usage
```

---

## WORKFLOW INPUT SCHEMAS

```typescript
// Niche Validation
{
  niche: string;            // Required, min 2 chars
  marketplace: "wildberries" | "ozon" | "yandex_market";
  include_reviews?: boolean; // Default: true
  max_competitors?: number;  // Default: 20, range 10-50
}

// Competitor Analysis
{
  competitor_url: string;   // Required, valid URL
  marketplace: "wildberries" | "ozon" | "yandex_market";
  include_price_history?: boolean;
  include_reviews?: boolean;
}

// Opportunity Finder
{
  budget: number;           // Required, min 50000 (rubles)
  category?: string;        // Optional
  risk_level: "conservative" | "moderate" | "aggressive";
  experience_level: "beginner" | "intermediate" | "advanced";
}
```

---

## ENVIRONMENT VARIABLES

```bash
# Backend (.env)
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/marketplace_ai
REDIS_URL=redis://localhost:6379/0
JWT_SECRET_KEY=your-secret-key-here
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=7
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
APIFY_API_TOKEN=apify_api_...

# Frontend (.env.local)
NEXT_PUBLIC_API_URL=http://localhost:8000
```

---

## WORKFLOW STEPS

### Niche Validation
1. ScrapeMarketDataStep (Apify)
2. AnalyzeTopPhotosStep (OpenAI Vision)
3. ScrapeReviewsStep (Apify)
4. AnalyzeSentimentStep (Claude Haiku)
5. SynthesizeReportStep (Claude Sonnet)

### Competitor Analysis
1. ExtractProductDataStep (Apify)
2. GetPriceHistoryStep (Apify)
3. AnalyzeListingPhotosStep (OpenAI Vision)
4. AnalyzeCompetitorReviewsStep (Claude Haiku)
5. GenerateVulnerabilityReportStep (Claude Sonnet)

### Opportunity Finder
1. IdentifyCategoriesStep (Claude Haiku)
2. ScanCategoriesStep (Apify)
3. CalculateOpportunityScoresStep (Claude Haiku)
4. DeepDiveTopOpportunitiesStep (Claude Haiku)
5. GenerateOpportunityReportStep (Claude Sonnet)

---

## API PRICING REFERENCE

| API | Model | Cost |
|-----|-------|------|
| Claude | Haiku | $0.25/$1.25 per 1M tokens |
| Claude | Sonnet | $3/$15 per 1M tokens |
| OpenAI | GPT-4o-mini | $0.15/$0.60 per 1M tokens |
| Apify | Actors | ~$0.05-0.10 per run |

---

## COMMON PATTERNS

### FastAPI Endpoint
```python
@router.post("/", response_model=WorkflowCreate)
async def create_workflow(
    input_data: NicheValidationInput,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    workflow = await workflow_service.create(db, current_user, input_data)
    return workflow
```

### Celery Task
```python
@celery_app.task(bind=True, max_retries=3)
def run_workflow_task(self, workflow_id: str):
    try:
        # Execute workflow
        pass
    except Exception as e:
        raise self.retry(exc=e, countdown=60)
```

### React Query Hook
```typescript
const { data, isLoading, error } = useQuery({
  queryKey: ['workflow', id],
  queryFn: () => api.getWorkflow(id),
  refetchInterval: status === 'running' ? 3000 : false,
});
```

### Zustand Store
```typescript
const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      login: async (email, password) => {
        const { user, token } = await api.login(email, password);
        set({ user, token });
      },
    }),
    { name: 'auth-storage' }
  )
);
```

---

## STATUS COLORS

| Status | Color | Tailwind Class |
|--------|-------|----------------|
| pending | Gray | `bg-gray-500` |
| queued | Blue | `bg-blue-500` |
| running | Blue (pulse) | `bg-blue-500 animate-pulse` |
| completed | Green | `bg-green-500` |
| failed | Red | `bg-red-500` |

---

## DAILY CHECKLIST

```
□ Pull latest code
□ Check for new environment variables
□ Run database migrations
□ Read today's prompts fully
□ Create branch for today's work
□ Implement features
□ Write tests
□ Test manually
□ Commit with clear messages
□ Push to repository
□ Update task tracker
```

---

## HELP RESOURCES

- FastAPI Docs: https://fastapi.tiangolo.com/
- Next.js Docs: https://nextjs.org/docs
- shadcn/ui: https://ui.shadcn.com/
- Tailwind CSS: https://tailwindcss.com/docs
- SQLAlchemy 2.0: https://docs.sqlalchemy.org/en/20/
- Celery: https://docs.celeryq.dev/
- Anthropic SDK: https://docs.anthropic.com/
- OpenAI SDK: https://platform.openai.com/docs

---

*Print this and keep it visible!*
