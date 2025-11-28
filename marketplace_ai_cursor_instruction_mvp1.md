# MARKETPLACE AI OS — MVP 1 Build Guide
## 10-Day Cursor Prompts for Junior Developers

---

# SUMMARY: 10-Day Sprint Overview

| Day | Focus | Deliverables |
|-----|-------|--------------|
| 1 | Database & Auth | Models, migrations, JWT auth, auth endpoints |
| 2 | Workflow Engine | Workflow models, base engine, Celery setup, API endpoints |
| 3 | API Integrations | Anthropic, OpenAI, Apify clients, caching layer |
| 4 | Niche Validation | Complete workflow with all steps and prompts |
| 5 | Other Workflows | Competitor Analysis + Opportunity Finder workflows |
| 6 | Frontend Setup | Next.js setup, auth pages, dashboard layout |
| 7 | Workflow UI | List page, creation flow, forms |
| 8 | Results Display | Detail page, results for all 3 workflow types |
| 9 | Polish | Error handling, usage tracking, settings |
| 10 | Deploy | Tests, documentation, production deployment |

---

# TIPS FOR JUNIOR DEVELOPERS

1. **Read the prompt fully before starting** - Each prompt has specific requirements
2. **Ask Cursor for clarification** - If something is unclear, ask
3. **Test as you go** - Don't wait until Day 10 to test
4. **Commit frequently** - Small, focused commits
5. **Don't skip error handling** - It's critical for production
6. **Check the types** - TypeScript will catch many bugs
7. **Look at the docs** - FastAPI, Next.js, shadcn/ui have great docs
8. **Use the previous day's code** - Build on what's already done

---

# COMMUNICATION

Daily standups should cover:
1. What was completed yesterday
2. What's planned for today
3. Any blockers or questions

End of each day:
1. Push all code to repository
2. Update task tracker
3. Note any deviations from plan

---

*Good luck building! 🚀*

# OVERVIEW

**What We're Building:** Research Agent MVP with 3 workflows
- Niche Validation
- Competitor Analysis  
- Opportunity Finder

**Tech Stack:**
- Backend: Python + FastAPI + PostgreSQL + Redis + Celery
- Frontend: Next.js + React + Tailwind + shadcn/ui
- Infrastructure: Docker + Docker Compose

**Team Structure:**
- Developer 1: Backend (Days 1-10)
- Developer 2: Frontend (Days 3-10)
- Both devs can work in parallel after Day 2

  

---

# PRE-WORK (Before Day 1)

## Prompt for Project Setup:

```
I'm building a SaaS platform called "Marketplace AI OS" for Russian marketplace sellers (Wildberries, Ozon).

Create the initial project structure with:

Backend (Python):
- FastAPI application
- Poetry for dependency management
- Project structure following clean architecture:
  /backend
    /app
      /api          # API routes
      /core         # Config, security, dependencies
      /db           # Database models and connection
      /schemas      # Pydantic schemas
      /services     # Business logic
      /workflows    # Workflow engine
      /integrations # External API clients
      /tasks        # Celery tasks
    /tests
    /alembic        # Migrations
    main.py
    pyproject.toml

Frontend (Next.js):
- Next.js 14 with App Router
- TypeScript
- Tailwind CSS
- shadcn/ui components
  /frontend
    /app
    /components
    /lib
    /hooks
    /types

Docker:
- docker-compose.yml with: api, celery_worker, postgres, redis
- Dockerfile for backend
- Dockerfile for frontend

Create all necessary config files:
- .env.example with all required variables
- .gitignore
- README.md with setup instructions

Initialize with basic health check endpoint that returns {"status": "ok"}.
```

---

# DAY 1: Database & Authentication

## Morning: Database Setup

### Prompt 1.1 — Database Models

```
I'm building a marketplace research platform. Create SQLAlchemy models for PostgreSQL.

Requirements:
1. Users table:
   - id (UUID, primary key)
   - email (unique, indexed)
   - password_hash
   - name
   - subscription_tier (enum: starter, seller, business, agency)
   - workflow_runs_used (int, default 0)
   - workflow_runs_limit (int, default 30)
   - created_at, updated_at timestamps
   - is_active (boolean)

2. API Keys table:
   - id (UUID)
   - user_id (foreign key)
   - key_hash
   - name
   - last_used_at
   - created_at

3. Brand Profiles table:
   - id (UUID)
   - user_id (foreign key)
   - name
   - description (text)
   - logo_url
   - colors (JSONB) - for primary/secondary colors
   - voice_tone (text) - brand voice description
   - created_at

Use SQLAlchemy 2.0 syntax with async support.
Include proper indexes for frequently queried fields.
Add relationship definitions between models.
Create an Alembic migration for these tables.

File structure:
/app/db/models/user.py
/app/db/models/brand.py
/app/db/base.py (Base class and imports)
/app/db/session.py (async session factory)
```

### Prompt 1.2 — Database Connection

```
Create the database connection setup for async PostgreSQL.

Requirements:
1. AsyncSession factory using SQLAlchemy 2.0
2. Dependency injection for FastAPI (get_db)
3. Connection pooling configuration
4. Health check function to verify DB connection

Use these settings from environment:
- DATABASE_URL (async postgresql URL)

Include proper error handling and logging.

Files:
/app/db/session.py
/app/core/config.py (add database settings)
```

## Afternoon: Authentication

### Prompt 1.3 — JWT Authentication

```
Implement JWT authentication for the FastAPI backend.

Requirements:
1. Password hashing with bcrypt (passlib)
2. JWT token generation and validation using python-jose
3. Access token (15 min expiry) and refresh token (7 days)
4. Token payload: user_id, email, subscription_tier, exp

Create these components:
1. /app/core/security.py:
   - hash_password(password: str) -> str
   - verify_password(plain: str, hashed: str) -> bool
   - create_access_token(data: dict) -> str
   - create_refresh_token(data: dict) -> str
   - decode_token(token: str) -> dict

2. /app/core/dependencies.py:
   - get_current_user() - FastAPI dependency that extracts and validates JWT
   - get_current_active_user() - ensures user is active
   - require_subscription(tiers: list) - decorator for tier-based access

3. Environment variables needed:
   - JWT_SECRET_KEY
   - JWT_ALGORITHM (default: HS256)
   - ACCESS_TOKEN_EXPIRE_MINUTES
   - REFRESH_TOKEN_EXPIRE_DAYS

Include proper error responses (401, 403) with clear messages.
```

### Prompt 1.4 — Auth Endpoints

```
Create authentication API endpoints.

Endpoints needed:
1. POST /api/auth/register
   - Input: email, password, name
   - Validate email format and password strength (min 8 chars)
   - Check email not already registered
   - Create user with 'starter' tier
   - Return access_token, refresh_token, user object

2. POST /api/auth/login
   - Input: email, password
   - Validate credentials
   - Return access_token, refresh_token, user object

3. POST /api/auth/refresh
   - Input: refresh_token
   - Validate refresh token
   - Return new access_token, refresh_token

4. GET /api/auth/me
   - Requires: valid access_token
   - Return current user object (exclude password_hash)

5. PUT /api/auth/me
   - Requires: valid access_token
   - Input: name (optional), current_password + new_password (optional)
   - Update user profile

Create Pydantic schemas for all request/response bodies.
Use proper HTTP status codes.
Add rate limiting consideration (add TODO comments).

Files:
/app/api/routes/auth.py
/app/schemas/auth.py
/app/schemas/user.py
/app/services/auth_service.py
```

### Prompt 1.5 — Day 1 Testing

```
Create tests for Day 1 functionality.

Test files:
1. /tests/test_auth.py:
   - test_register_success
   - test_register_duplicate_email
   - test_register_weak_password
   - test_login_success
   - test_login_wrong_password
   - test_login_nonexistent_user
   - test_refresh_token
   - test_get_current_user
   - test_protected_route_without_token

2. /tests/conftest.py:
   - pytest fixtures for test database
   - test client fixture
   - authenticated user fixture

Use pytest-asyncio for async tests.
Use httpx for async test client.
Create a test database that resets between tests.
```

---

# DAY 2: Workflow Engine & Database

## Morning: Workflow Models

### Prompt 2.1 — Workflow Database Models

```
Create database models for the workflow system.

Models needed:

1. Workflows table:
   - id (UUID)
   - user_id (FK to users)
   - brand_id (FK to brand_profiles, nullable)
   - workflow_type (enum: niche_validation, competitor_analysis, opportunity_finder)
   - status (enum: pending, queued, running, completed, failed, cancelled)
   - input_data (JSONB) - workflow input parameters
   - output_data (JSONB, nullable) - workflow results
   - error_message (text, nullable)
   - started_at (timestamp, nullable)
   - completed_at (timestamp, nullable)
   - created_at
   - Indexes: user_id, status, created_at

2. Workflow Steps table:
   - id (UUID)
   - workflow_id (FK)
   - step_name (varchar 100)
   - step_order (int)
   - status (enum: pending, running, completed, failed, skipped)
   - input_data (JSONB, nullable)
   - output_data (JSONB, nullable)
   - api_provider (varchar 50) - claude, openai, apify, etc
   - tokens_used (int, nullable)
   - api_cost (decimal 10,6, nullable)
   - error_message (text, nullable)
   - started_at, completed_at (timestamps)

3. Research Cache table (for caching API responses):
   - id (UUID)
   - cache_key (varchar 255, unique, indexed)
   - data_type (varchar 50) - niche_data, competitor_data, reviews
   - data (JSONB)
   - expires_at (timestamp)
   - created_at

Create Alembic migration.
Add relationships to User model.

Files:
/app/db/models/workflow.py
/app/db/models/cache.py
Update /app/db/base.py
```

### Prompt 2.2 — Workflow Schemas

```
Create Pydantic schemas for workflow API requests and responses.

Schemas needed:

1. Workflow Input Schemas:
   - NicheValidationInput:
     - niche: str (required, min 2 chars)
     - marketplace: Literal["wildberries", "ozon", "yandex_market"]
     - include_reviews: bool = True
     - max_competitors: int = 20

   - CompetitorAnalysisInput:
     - competitor_url: str (required, valid URL)
     - marketplace: Literal["wildberries", "ozon", "yandex_market"]
     - include_price_history: bool = True
     - include_reviews: bool = True

   - OpportunityFinderInput:
     - budget: int (required, min 50000) - in rubles
     - category: str | None = None
     - risk_level: Literal["conservative", "moderate", "aggressive"]
     - experience_level: Literal["beginner", "intermediate", "advanced"]

2. Workflow Response Schemas:
   - WorkflowCreate (response after creating):
     - id, workflow_type, status, created_at
   
   - WorkflowStatus:
     - id, status, progress_percent, current_step, error_message
   
   - WorkflowDetail:
     - All fields including input_data, output_data, steps
   
   - WorkflowList:
     - Paginated list with id, type, status, created_at, completed_at

3. Step schemas for progress tracking

Use proper validation with Field() and validators.
Add examples for OpenAPI documentation.

Files:
/app/schemas/workflow.py
```

## Afternoon: Workflow Engine Base

### Prompt 2.3 — Workflow Engine Architecture

```
Create the base workflow engine architecture.

Design pattern: Each workflow is a sequence of steps. Each step:
- Has a name and order
- Calls an external API or performs computation
- Returns output that feeds into the next step
- Tracks cost and token usage
- Can be retried on failure

Create these base classes:

1. /app/workflows/base.py:
   
   class WorkflowStep(ABC):
       name: str
       api_provider: str
       max_retries: int = 3
       
       @abstractmethod
       async def execute(self, input_data: dict, context: WorkflowContext) -> dict:
           pass
       
       @abstractmethod
       def estimate_cost(self, input_data: dict) -> float:
           pass

   class Workflow(ABC):
       name: str
       workflow_type: str
       steps: list[WorkflowStep]
       
       async def run(self, input_data: dict, workflow_id: str) -> dict:
           # Iterate through steps, track progress, handle errors
           pass

   class WorkflowContext:
       workflow_id: str
       user_id: str
       db_session: AsyncSession
       accumulated_data: dict  # Data passed between steps
       total_cost: float
       
2. /app/workflows/registry.py:
   - Registry to map workflow_type string to Workflow class
   - get_workflow(workflow_type: str) -> Workflow

Include:
- Proper error handling with custom exceptions
- Progress tracking (update workflow_steps in DB)
- Cost accumulation
- Logging at each step
```

### Prompt 2.4 — Celery Setup

```
Set up Celery for async workflow execution.

Requirements:

1. /app/tasks/celery_app.py:
   - Celery app configuration
   - Redis as broker and backend
   - Task serialization settings
   - Task time limits (10 min max)
   - Retry configuration

2. /app/tasks/workflow_tasks.py:
   - run_workflow_task(workflow_id: str, workflow_type: str, input_data: dict)
     - Fetches workflow from DB
     - Updates status to 'running'
     - Executes workflow
     - Updates status to 'completed' or 'failed'
     - Stores output_data or error_message

3. Environment variables:
   - REDIS_URL
   - CELERY_TASK_TIME_LIMIT

4. Update docker-compose.yml:
   - Add celery_worker service
   - Ensure it has access to all env vars

Include:
- Proper async handling (run_sync wrapper for Celery)
- Database session management in tasks
- Error handling and retry logic
- Task status tracking
```

### Prompt 2.5 — Workflow API Endpoints

```
Create API endpoints for workflow management.

Endpoints:

1. POST /api/workflows/research/niche-validation
   - Auth required
   - Check user has remaining workflow runs
   - Validate input with NicheValidationInput schema
   - Create workflow record in DB
   - Queue Celery task
   - Return WorkflowCreate response
   - Decrement user's workflow_runs_used

2. POST /api/workflows/research/competitor-analysis
   - Same pattern as above with CompetitorAnalysisInput

3. POST /api/workflows/research/opportunity-finder
   - Same pattern with OpportunityFinderInput

4. GET /api/workflows
   - Auth required
   - Query params: status (optional), limit (default 20), offset
   - Return paginated WorkflowList for current user

5. GET /api/workflows/{workflow_id}
   - Auth required
   - Verify workflow belongs to user
   - Return WorkflowDetail with all data

6. GET /api/workflows/{workflow_id}/status
   - Auth required
   - Lightweight endpoint for polling
   - Return WorkflowStatus only

7. DELETE /api/workflows/{workflow_id}
   - Auth required
   - Only allow if status is pending/queued
   - Cancel and delete workflow

Create service layer for business logic.

Files:
/app/api/routes/workflows.py
/app/services/workflow_service.py
```

---

# DAY 3: External API Integrations

## Morning: API Client Base & Anthropic

### Prompt 3.1 — API Client Base Class

```
Create a base class for all external API integrations.

Requirements:

1. /app/integrations/base.py:
   
   class APIClient(ABC):
       """Base class for external API clients"""
       
       provider_name: str
       
       def __init__(self):
           self.logger = logging.getLogger(f"api.{self.provider_name}")
       
       async def _make_request(
           self,
           method: str,
           url: str,
           **kwargs
       ) -> dict:
           """Make HTTP request with retry logic, logging, error handling"""
           pass
       
       @abstractmethod
       def estimate_cost(self, **kwargs) -> float:
           """Estimate cost for an operation"""
           pass

Features to include:
- Automatic retries with exponential backoff (3 attempts)
- Request/response logging (sanitize sensitive data)
- Timeout configuration
- Rate limiting awareness
- Custom exceptions: APIError, RateLimitError, AuthenticationError
- Cost tracking

2. /app/integrations/exceptions.py:
   - Custom exception classes for API errors

Use httpx for async HTTP requests.
```

### Prompt 3.2 — Anthropic (Claude) Integration

```
Create the Anthropic API client for Claude models.

File: /app/integrations/anthropic_client.py

Requirements:

1. AnthropicClient class:
   - Uses official anthropic Python SDK
   - Supports both Haiku and Sonnet models
   
   Methods:
   - async generate(
       prompt: str,
       system: str = "",
       model: str = "claude-3-5-haiku-20241022",
       max_tokens: int = 1000,
       temperature: float = 0.7
     ) -> tuple[str, dict]  # Returns (response_text, usage_info)
   
   - async generate_json(
       prompt: str,
       system: str = "",
       response_schema: dict = None,  # Expected JSON structure
       model: str = "claude-3-5-haiku-20241022"
     ) -> dict  # Parsed JSON response
   
   - estimate_cost(input_tokens: int, output_tokens: int, model: str) -> float

2. Pricing constants:
   PRICING = {
       "claude-3-5-haiku-20241022": {"input": 0.25, "output": 1.25},  # per 1M
       "claude-3-5-sonnet-20241022": {"input": 3.0, "output": 15.0},
   }

3. Error handling:
   - Handle rate limits with retry
   - Handle API errors gracefully
   - Log all requests with token counts

4. JSON parsing:
   - Strip markdown code blocks from response
   - Validate JSON structure
   - Retry once if JSON parsing fails

Environment variable: ANTHROPIC_API_KEY
```

### Prompt 3.3 — OpenAI (Vision) Integration

```
Create the OpenAI API client for GPT-4o Vision.

File: /app/integrations/openai_client.py

Requirements:

1. OpenAIClient class:
   - Uses official openai Python SDK
   
   Methods:
   - async analyze_image(
       image_url: str,
       prompt: str,
       model: str = "gpt-4o-mini"
     ) -> tuple[str, dict]  # Returns (response_text, usage_info)
   
   - async analyze_image_base64(
       image_base64: str,
       prompt: str,
       media_type: str = "image/jpeg"
     ) -> tuple[str, dict]
   
   - async analyze_images_batch(
       images: list[str],  # URLs
       prompt: str,
       model: str = "gpt-4o-mini"
     ) -> list[tuple[str, dict]]
   
   - estimate_cost(input_tokens: int, output_tokens: int, model: str) -> float

2. Pricing constants for vision models

3. Image handling:
   - Validate image URLs are accessible
   - Handle both URL and base64 inputs
   - Resize/compress if needed (add TODO)

4. JSON response parsing (same as Anthropic)

Environment variable: OPENAI_API_KEY
```

## Afternoon: Apify & Caching

### Prompt 3.4 — Apify Scraping Integration

```
Create the Apify API client for marketplace scraping.

File: /app/integrations/apify_client.py

Requirements:

1. ApifyClient class:
   - Uses apify-client Python SDK
   
   Methods:
   - async run_actor(
       actor_id: str,
       input_data: dict,
       timeout_secs: int = 300,
       memory_mbytes: int = 1024
     ) -> list[dict]  # Returns scraped items
   
   - async scrape_wildberries_search(
       query: str,
       max_results: int = 50,
       sort_by: str = "popular"
     ) -> list[dict]  # Product listings
   
   - async scrape_wildberries_product(
       product_url: str
     ) -> dict  # Single product details
   
   - async scrape_product_reviews(
       product_url: str,
       max_reviews: int = 100
     ) -> list[dict]  # Reviews
   
   - async scrape_ozon_search(
       query: str,
       max_results: int = 50
     ) -> list[dict]

2. Actor IDs (use placeholders, will configure later):
   ACTORS = {
       "wildberries_search": "your-wb-search-actor",
       "wildberries_product": "your-wb-product-actor",
       "ozon_search": "your-ozon-actor",
       "reviews": "your-reviews-actor",
   }

3. Data normalization:
   - Create consistent output schema regardless of actor
   - Normalize prices to kopeks
   - Parse dates to datetime objects

4. Error handling:
   - Handle actor timeout
   - Handle empty results
   - Handle rate limits

Environment variable: APIFY_API_TOKEN
```

### Prompt 3.5 — Caching Layer

```
Create a caching layer for expensive API calls.

File: /app/services/cache_service.py

Requirements:

1. CacheService class:
   
   Methods:
   - async get(cache_key: str) -> dict | None
     - Check if cache exists and not expired
     - Return cached data or None
   
   - async set(
       cache_key: str,
       data: dict,
       data_type: str,
       ttl_hours: int = 24
     ) -> None
     - Store data with expiration
   
   - async get_or_fetch(
       cache_key: str,
       fetch_func: Callable,
       data_type: str,
       ttl_hours: int = 24
     ) -> dict
     - Get from cache or call fetch_func and cache result
   
   - async invalidate(cache_key: str) -> None
   
   - generate_cache_key(prefix: str, **params) -> str
     - Create deterministic cache key from parameters

2. Cache key patterns:
   - niche:{marketplace}:{query_hash}
   - competitor:{marketplace}:{url_hash}
   - reviews:{product_id}:{page}

3. Use database (research_cache table) for persistence
   - Also add Redis caching for hot data (optional, add TODO)

4. Automatic cleanup:
   - Create Celery beat task to clean expired cache entries daily

Include logging for cache hits/misses.
```

---

# DAY 4: Niche Validation Workflow

## Full Day: Implement Complete Workflow

### Prompt 4.1 — Niche Validation Steps (Part 1)

```
Create the first steps of the Niche Validation workflow.

File: /app/workflows/research/niche_validation.py

Step 1: ScrapeMarketDataStep
- Uses ApifyClient to scrape top 50 products for the niche
- Extracts: product_id, title, price, rating, review_count, seller_name, image_url
- Calculates: price_min, price_max, price_avg, price_median
- Output: {products: [...], price_stats: {...}, total_results: int}

Step 2: AnalyzeTopPhotosStep
- Takes top 10 products by sales/rating
- Uses OpenAI Vision to analyze each product's main image
- Prompt should evaluate: background, lighting, infographics, quality
- Output: {photo_analyses: [{product_id, analysis: {...}}, ...]}

Step 3: ScrapeReviewsStep
- Scrapes reviews for top 5 products (100 reviews each max)
- Uses ApifyClient
- Output: {reviews: [{product_id, reviews: [...]}], total_reviews: int}

Implementation requirements:
- Each step extends WorkflowStep base class
- Include estimate_cost() for each step
- Handle errors gracefully (if scraping fails, continue with partial data)
- Log progress at each step
- Update workflow_steps in database

Create Pydantic models for step inputs/outputs.
```

### Prompt 4.2 — Niche Validation Steps (Part 2)

```
Create the analysis steps of the Niche Validation workflow.

Continuing in: /app/workflows/research/niche_validation.py

Step 4: AnalyzeSentimentStep
- Takes all scraped reviews
- Uses Claude Haiku to analyze sentiment and extract insights
- Prompt should identify: top complaints, top praises, feature requests, quality issues

System prompt:
"You are a customer sentiment analyst for Russian e-commerce. Analyze product reviews to extract actionable insights. Respond in JSON format."

User prompt template (create detailed prompt):
- Include all reviews
- Request structured output with: sentiment_distribution, top_complaints, top_praises, etc.

Output: {sentiment_analysis: {...}}

Step 5: SynthesizeReportStep
- Takes ALL previous step outputs
- Uses Claude Sonnet (need quality for final report)
- Generates comprehensive niche validation report

System prompt:
"You are a senior marketplace strategist with expertise in Wildberries, Ozon, and Yandex Market. Provide data-driven, actionable analysis."

User prompt template:
- Include all gathered data
- Request structured report with: executive_summary, market_overview, competition_analysis, opportunity_score, go_no_go recommendation, action_plan

Output: Final report JSON

Create the NicheValidationWorkflow class that chains all 5 steps.
Register it in the workflow registry.
```

### Prompt 4.3 — Niche Validation Prompts

```
Create the prompt templates for Niche Validation workflow.

File: /app/workflows/research/prompts/niche_validation.py

Create these prompt templates as Python string constants:

1. PHOTO_ANALYSIS_PROMPT
   - Input placeholders: {product_category}
   - Instructs GPT-4o to analyze marketplace product photo
   - Requests JSON output with: background, lighting, composition, infographics, quality_score, strengths, weaknesses

2. SENTIMENT_ANALYSIS_SYSTEM_PROMPT
   - Role definition for sentiment analyst
   - Instructions for Russian e-commerce context

3. SENTIMENT_ANALYSIS_USER_PROMPT
   - Input placeholders: {review_count}, {product_category}, {marketplace}, {reviews_text}
   - Detailed analysis request
   - Expected JSON structure

4. SYNTHESIS_SYSTEM_PROMPT
   - Role definition for marketplace strategist
   - Context about Russian marketplaces

5. SYNTHESIS_USER_PROMPT
   - Input placeholders: {niche}, {marketplace}, {sellers_data}, {products_data}, {price_stats}, {photo_analyses}, {sentiment_analysis}
   - Full report structure request
   - GO/NO-GO recommendation requirement

Make prompts production-quality:
- Clear instructions
- Explicit JSON structure requirements
- Examples where helpful
- Error prevention (e.g., "Ensure all fields are present")
```

### Prompt 4.4 — Testing Niche Validation

```
Create tests for the Niche Validation workflow.

File: /tests/workflows/test_niche_validation.py

Tests needed:

1. Unit tests for each step:
   - test_scrape_market_data_step (mock Apify response)
   - test_analyze_photos_step (mock OpenAI response)
   - test_scrape_reviews_step (mock Apify response)
   - test_analyze_sentiment_step (mock Claude response)
   - test_synthesize_report_step (mock Claude response)

2. Integration test:
   - test_niche_validation_workflow_complete
   - Uses mocked API responses
   - Verifies all steps execute in order
   - Verifies final output structure

3. Edge case tests:
   - test_workflow_with_no_products_found
   - test_workflow_with_no_reviews
   - test_workflow_with_api_error_recovery

Create fixtures:
- Mock API responses for each external service
- Sample input data
- Expected output structures

Use pytest-mock for mocking.
```

---

# DAY 5: Competitor Analysis & Opportunity Finder

## Morning: Competitor Analysis Workflow

### Prompt 5.1 — Competitor Analysis Workflow

```
Create the Competitor Analysis workflow.

File: /app/workflows/research/competitor_analysis.py

This workflow analyzes a single competitor in depth.

Steps:

1. ExtractProductDataStep
   - Parse the competitor URL to get product ID
   - Use Apify to scrape full product details
   - Extract: title, description, price, rating, review_count, seller_info, images, category
   - Output: {product: {...}}

2. GetPriceHistoryStep
   - Use Apify or MPStats API (add TODO for MPStats integration)
   - Get 90-day price history
   - Calculate: avg_price, min_price, max_price, discount_frequency, price_volatility
   - Output: {price_history: [...], price_stats: {...}}

3. AnalyzeListingPhotosStep
   - Analyze all product images (up to 10)
   - Use OpenAI Vision
   - Score each image, identify strengths/weaknesses
   - Output: {photo_analysis: {...}}

4. AnalyzeCompetitorReviewsStep
   - Scrape up to 200 reviews
   - Use Claude Haiku for sentiment analysis
   - Focus on: complaints, quality issues, what customers love
   - Output: {review_analysis: {...}}

5. GenerateVulnerabilityReportStep
   - Use Claude Sonnet
   - Synthesize all data into competitive intelligence report
   - Identify: strengths, weaknesses, vulnerabilities, how to beat them
   - Output: Full competitor report

Create prompt templates in /app/workflows/research/prompts/competitor_analysis.py

Register workflow in registry.
```

## Afternoon: Opportunity Finder Workflow

### Prompt 5.2 — Opportunity Finder Workflow

```
Create the Opportunity Finder workflow.

File: /app/workflows/research/opportunity_finder.py

This workflow finds product opportunities based on user's budget and preferences.

Steps:

1. IdentifyCategoriesStep
   - If category not specified, use Claude to suggest 5 categories based on budget and experience
   - If category specified, generate 3 sub-categories to explore
   - Output: {categories_to_analyze: [...]}

2. ScanCategoriesStep
   - For each category (up to 5), scrape top 30 products
   - Use Apify
   - Collect: prices, ratings, review counts, seller counts
   - Output: {category_data: [{category, products: [...], stats: {...}}, ...]}

3. CalculateOpportunityScoresStep
   - For each category, calculate:
     - demand_score (based on review counts, search volume proxy)
     - competition_score (number of sellers, concentration)
     - margin_potential (based on typical pricing)
     - entry_barrier (based on avg ratings, review counts to compete)
   - Rank categories by overall opportunity score
   - Output: {scored_categories: [...]}

4. DeepDiveTopOpportunitiesStep
   - Take top 3 categories
   - For each, identify specific product opportunities
   - Scrape reviews for top products to understand gaps
   - Use Claude Haiku
   - Output: {opportunities: [{category, products: [...], insights: {...}}, ...]}

5. GenerateOpportunityReportStep
   - Use Claude Sonnet
   - Create ranked list of opportunities with:
     - Investment required
     - Expected ROI
     - Risk assessment
     - Step-by-step entry plan
   - Output: Final opportunity report

Create prompt templates.
Register workflow.
```

### Prompt 5.3 — Opportunity Finder Prompts

```
Create prompts for Opportunity Finder workflow.

File: /app/workflows/research/prompts/opportunity_finder.py

Prompts needed:

1. CATEGORY_SUGGESTION_PROMPT
   - Input: {budget}, {risk_level}, {experience_level}
   - Ask Claude to suggest 5 marketplace categories
   - Consider: budget constraints, competition level, margin potential
   - Output: list of categories with reasoning

2. OPPORTUNITY_SCORING_PROMPT
   - Input: {category_data} with product stats
   - Ask Claude to calculate opportunity scores
   - Explain scoring methodology
   - Output: scored categories with reasoning

3. DEEP_DIVE_ANALYSIS_PROMPT
   - Input: {category}, {top_products}, {reviews}
   - Identify specific product opportunities within category
   - Find gaps in market (unmet needs, quality issues)
   - Output: list of specific opportunities

4. FINAL_REPORT_PROMPT
   - Input: all analysis data, {user_budget}, {user_preferences}
   - Generate comprehensive opportunity report
   - Include financial projections
   - Provide actionable next steps
   - Output: structured report JSON

Make prompts specific to Russian marketplace context.
Include ruble amounts, local examples.
```

---

# DAY 6: Frontend Setup & Authentication UI

## Full Day: Frontend Foundation

### Prompt 6.1 — Next.js Project Setup

```
Set up the Next.js frontend project with all necessary configurations.

Requirements:

1. Next.js 14 with App Router
2. TypeScript configuration
3. Tailwind CSS with custom theme:
   - Colors matching our brand (orange #FF6B35, teal #4ECDC4, purple #9B5DE5)
   - Dark mode as default
   - Custom fonts (Inter for body, Space Grotesk for headings)

4. Install and configure shadcn/ui:
   - Button, Input, Card, Dialog, Toast, Dropdown, Table, Badge, Progress
   - Dark theme

5. Project structure:
   /app
     /(auth)
       /login/page.tsx
       /register/page.tsx
       layout.tsx
     /(dashboard)
       /page.tsx
       /workflows/page.tsx
       /workflows/[id]/page.tsx
       /workflows/new/page.tsx
       layout.tsx
     /api (if needed for BFF)
     layout.tsx
     globals.css
   /components
     /ui (shadcn components)
     /auth
     /dashboard
     /workflows
     /common
   /lib
     /api.ts (API client)
     /auth.ts (auth utilities)
     /utils.ts
   /hooks
   /types
   /stores (Zustand stores)

6. Environment variables:
   - NEXT_PUBLIC_API_URL

Create basic layout with placeholder content.
```

### Prompt 6.2 — API Client & Auth Store

```
Create the API client and authentication state management.

Files:

1. /lib/api.ts - Axios-based API client:
   - Base URL from environment
   - Request interceptor to add JWT token
   - Response interceptor for 401 handling (redirect to login)
   - Typed request methods: get, post, put, delete
   - Error handling with toast notifications

2. /types/index.ts - TypeScript types:
   - User interface
   - Workflow interfaces (list item, detail, create request)
   - API response wrapper types
   - Form input types

3. /stores/auth-store.ts - Zustand store:
   - State: user, accessToken, refreshToken, isLoading, isAuthenticated
   - Actions: login, register, logout, refreshToken, loadFromStorage
   - Persist to localStorage (tokens only, not user object)

4. /hooks/use-auth.ts:
   - Custom hook wrapping auth store
   - Provides: user, isAuthenticated, login, register, logout
   - Handles loading states

5. /lib/auth.ts:
   - getStoredToken() 
   - setStoredToken()
   - clearStoredToken()
   - isTokenExpired()

Include proper TypeScript types throughout.
```

### Prompt 6.3 — Auth Pages UI

```
Create the authentication pages with beautiful UI.

Files:

1. /app/(auth)/layout.tsx:
   - Centered layout for auth pages
   - Background with subtle gradient/pattern
   - Logo and branding

2. /app/(auth)/login/page.tsx:
   - Email and password inputs
   - "Remember me" checkbox
   - Submit button with loading state
   - Link to register page
   - Error message display
   - Redirect to dashboard on success

3. /app/(auth)/register/page.tsx:
   - Name, email, password, confirm password inputs
   - Password strength indicator
   - Terms acceptance checkbox
   - Submit button with loading state
   - Link to login page
   - Success message and redirect

4. /components/auth/login-form.tsx:
   - Form component with react-hook-form
   - Zod validation
   - Loading states
   - Error handling

5. /components/auth/register-form.tsx:
   - Same pattern as login form
   - Password confirmation validation

Design requirements:
- Dark theme
- Clean, modern look
- Brand colors as accents
- Responsive (mobile-first)
- Smooth animations on interactions
```

### Prompt 6.4 — Protected Routes & Dashboard Layout

```
Create protected route wrapper and dashboard layout.

Files:

1. /components/auth/protected-route.tsx:
   - Wrapper component that checks authentication
   - Redirects to /login if not authenticated
   - Shows loading spinner while checking
   - Refreshes token if needed

2. /app/(dashboard)/layout.tsx:
   - Sidebar navigation (collapsible on mobile)
   - Header with user menu
   - Main content area
   - Uses ProtectedRoute wrapper

3. /components/dashboard/sidebar.tsx:
   - Logo at top
   - Navigation items:
     - Dashboard (home icon)
     - Workflows (play icon)
     - Assets (folder icon) - coming soon badge
     - Settings (gear icon)
   - Usage indicator at bottom (X/30 workflows used)
   - Collapse button

4. /components/dashboard/header.tsx:
   - Page title (dynamic)
   - Search bar (placeholder for now)
   - Notifications bell (placeholder)
   - User dropdown menu:
     - User name and email
     - Subscription tier badge
     - Settings link
     - Logout button

5. /components/dashboard/user-menu.tsx:
   - Dropdown with user info and actions

Design: Dark theme, brand colors for accents, clean and professional.
```

---

# DAY 7: Workflow UI - List & Creation

## Morning: Workflow List Page

### Prompt 7.1 — Workflows List Page

```
Create the workflows list page showing all user's workflows.

Files:

1. /app/(dashboard)/workflows/page.tsx:
   - Page title "My Workflows"
   - "New Workflow" button (links to /workflows/new)
   - Filters: status dropdown, workflow type dropdown
   - Workflows table/grid
   - Pagination
   - Empty state when no workflows

2. /components/workflows/workflow-list.tsx:
   - Fetches workflows from API
   - Loading skeleton while fetching
   - Error state with retry button

3. /components/workflows/workflow-card.tsx:
   - Card display for each workflow
   - Shows: type icon, title (niche/competitor name), status badge, created date
   - Status badge colors: pending (gray), running (blue pulse), completed (green), failed (red)
   - Click to view details
   - Quick actions menu (view, delete)

4. /components/workflows/workflow-table.tsx:
   - Alternative table view
   - Columns: Type, Name/Input, Status, Created, Duration, Actions
   - Sortable columns
   - Row click to view details

5. /hooks/use-workflows.ts:
   - Custom hook for fetching workflows
   - Includes: data, isLoading, error, refetch
   - Uses SWR or React Query for caching

6. /stores/workflow-store.ts:
   - Zustand store for workflow state
   - Selected workflow, filters, pagination

Add toggle between card and table view.
Implement real-time status updates (polling every 5s for running workflows).
```

### Prompt 7.2 — New Workflow Page

```
Create the new workflow creation page.

Files:

1. /app/(dashboard)/workflows/new/page.tsx:
   - Page title "Create New Workflow"
   - Workflow type selection cards
   - Step indicator (1. Select Type → 2. Configure → 3. Review)

2. /components/workflows/workflow-type-selector.tsx:
   - Three cards for workflow types:
     
     Niche Validation:
     - Icon: magnifying glass + chart
     - Description: "Validate a product niche before entering the market"
     - Estimated time: 2-5 minutes
     
     Competitor Analysis:
     - Icon: users + eye
     - Description: "Deep dive into a specific competitor's strategy"
     - Estimated time: 3-5 minutes
     
     Opportunity Finder:
     - Icon: lightbulb + rocket
     - Description: "Find profitable product opportunities based on your budget"
     - Estimated time: 5-10 minutes
   
   - Each card is clickable
   - Selected state with border highlight

3. /components/workflows/forms/niche-validation-form.tsx:
   - Form fields:
     - Niche/Product keyword (text input, required)
     - Marketplace (select: Wildberries, Ozon, Yandex Market)
     - Include reviews analysis (toggle, default true)
     - Max competitors to analyze (slider: 10-50, default 20)
   - Form validation with Zod
   - Submit button

4. /components/workflows/forms/competitor-analysis-form.tsx:
   - Form fields:
     - Competitor product URL (URL input with validation)
     - Marketplace (auto-detect from URL or select)
     - Include price history (toggle)
     - Include reviews (toggle)

5. /components/workflows/forms/opportunity-finder-form.tsx:
   - Form fields:
     - Starting budget in rubles (number input with formatting)
     - Category preference (optional text or select from common categories)
     - Risk tolerance (radio: Conservative, Moderate, Aggressive)
     - Experience level (radio: Beginner, Intermediate, Advanced)

Each form shows estimated API cost and time.
```

## Afternoon: Workflow Creation Flow

### Prompt 7.3 — Workflow Creation Confirmation & Submission

```
Create the workflow creation confirmation and submission flow.

Files:

1. /components/workflows/workflow-review.tsx:
   - Summary of selected options
   - Estimated cost display
   - Estimated time display
   - User's remaining workflow runs (X of 30)
   - Warning if low on runs
   - "Create Workflow" button
   - "Back" button to edit

2. /components/workflows/workflow-created-modal.tsx:
   - Success modal after workflow created
   - Shows workflow ID
   - Options:
     - "View Progress" - goes to workflow detail page
     - "Create Another" - resets form
     - "Go to Dashboard" - goes to workflows list

3. /app/(dashboard)/workflows/new/[type]/page.tsx:
   - Dynamic route for each workflow type
   - Renders appropriate form based on type
   - Handles form submission
   - Shows confirmation before submitting
   - Calls API to create workflow
   - Shows success modal

4. /lib/api/workflows.ts:
   - createNicheValidation(input: NicheValidationInput)
   - createCompetitorAnalysis(input: CompetitorAnalysisInput)
   - createOpportunityFinder(input: OpportunityFinderInput)
   - getWorkflows(filters)
   - getWorkflow(id)
   - deleteWorkflow(id)

5. /hooks/use-create-workflow.ts:
   - Mutation hook for creating workflows
   - Handles loading, error, success states
   - Invalidates workflow list cache on success

Implement proper error handling with user-friendly messages.
Add analytics tracking (console.log for now, add TODO for real analytics).
```

---

# DAY 8: Workflow Detail & Results Display

## Full Day: Workflow Detail Page

### Prompt 8.1 — Workflow Detail Page Structure

```
Create the workflow detail page showing status and results.

Files:

1. /app/(dashboard)/workflows/[id]/page.tsx:
   - Fetches workflow by ID
   - Shows different views based on status:
     - Pending/Running: Progress view
     - Completed: Results view
     - Failed: Error view with retry option
   - Breadcrumb navigation
   - Back button

2. /components/workflows/workflow-progress.tsx:
   - For pending/running workflows
   - Shows current step being executed
   - Progress bar (percentage or step-based)
   - Step list with status indicators:
     - Completed steps (green check)
     - Current step (blue spinner)
     - Pending steps (gray)
   - Estimated time remaining
   - Cancel button (if pending)
   - Auto-refresh every 3 seconds

3. /components/workflows/workflow-error.tsx:
   - For failed workflows
   - Shows error message
   - Shows which step failed
   - Retry button (creates new workflow with same inputs)
   - Contact support link

4. /hooks/use-workflow-detail.ts:
   - Fetches single workflow
   - Polling for status updates when running
   - Stops polling when completed/failed

Add skeleton loading states.
Handle 404 (workflow not found).
```

### Prompt 8.2 — Niche Validation Results Display

```
Create the results display for Niche Validation workflow.

File: /components/workflows/results/niche-validation-results.tsx

This component displays the full niche validation report.

Sections:

1. Executive Summary Card:
   - Large card at top
   - GO / NO-GO badge (green or red)
   - Opportunity score (1-10) with visual indicator
   - 2-3 sentence summary
   - Confidence level badge

2. Market Overview Section:
   - Stats cards in grid:
     - Estimated monthly revenue
     - Estimated monthly units
     - Average price
     - Number of sellers
   - Market growth trend indicator
   - Seasonality notes

3. Competition Analysis Section:
   - Competition intensity gauge (low/medium/high/very high)
   - Top players table:
     - Seller name
     - Estimated revenue
     - Strengths
     - Weaknesses
   - Barriers to entry list

4. Customer Insights Section:
   - Top complaints (list with frequency bars)
   - Top praises (list with frequency bars)
   - Unmet needs (highlighted)
   - Price sensitivity indicator

5. Visual Standards Section:
   - Photo quality benchmark description
   - Common styles used
   - Differentiation opportunities

6. Financial Projections Section:
   - Recommended price point
   - Estimated margin
   - Break-even analysis
   - 6-month and 12-month projections

7. Action Plan Section:
   - Numbered next steps
   - Week 1 priorities
   - Success metrics to track

8. Download/Export:
   - "Download as PDF" button (placeholder)
   - "Copy to clipboard" for key stats

Use charts where appropriate (simple bar charts, gauges).
Make it printable-friendly.
```

### Prompt 8.3 — Competitor Analysis Results Display

```
Create the results display for Competitor Analysis workflow.

File: /components/workflows/results/competitor-analysis-results.tsx

Sections:

1. Competitor Overview Card:
   - Product image (if available)
   - Product title
   - Seller name
   - Current price
   - Rating with stars
   - Review count
   - Market position badge

2. Vulnerability Score:
   - Large visual score (1-10)
   - "How easy to beat" interpretation
   - Key vulnerability highlighted

3. SWOT-style Analysis:
   - Strengths grid (with icons)
   - Weaknesses grid (with icons)
   - Each item has "difficulty to replicate" or "exploitation opportunity"

4. Pricing Strategy Section:
   - Price history chart (line chart, 90 days)
   - Discount frequency
   - Price position (premium/mid/budget)
   - Margin estimate

5. Listing Quality Audit:
   - Photo score with breakdown
   - Title optimization score
   - Description quality score
   - Missing elements list

6. Customer Satisfaction:
   - Sentiment pie chart
   - Common complaints list
   - Unaddressed issues (opportunities)

7. How to Beat Them Section:
   - Primary strategy recommendation
   - Specific differentiation angle
   - Recommended price point
   - Quality improvements needed
   - Timeline to compete

8. Risk Assessment:
   - Likelihood of retaliation
   - Expected response
   - Mitigation strategy

Use cards with color coding for positive/negative items.
```

### Prompt 8.4 — Opportunity Finder Results Display

```
Create the results display for Opportunity Finder workflow.

File: /components/workflows/results/opportunity-finder-results.tsx

Sections:

1. Summary Card:
   - "X opportunities found in Y categories"
   - Top pick highlighted
   - User's budget displayed

2. Top Opportunities List:
   - Ranked cards (1-5)
   - Each card shows:
     - Rank badge
     - Product/category name
     - Opportunity score (visual)
     - Investment required
     - Expected monthly revenue
     - Risk level badge
     - "View Details" expand button

3. Opportunity Detail (expandable):
   - Market data stats
   - Why this opportunity (explanation)
   - Financial estimates:
     - Recommended price
     - Estimated COGS
     - Estimated margin
     - Break-even timeline
   - Risks list
   - Success requirements
   - Quick start guide (numbered steps)

4. Categories Analyzed:
   - Collapsible section showing all scanned categories
   - Scores for each
   - Why not recommended (for lower-ranked)

5. Opportunities to Avoid:
   - Warning section
   - Products/categories to skip
   - Reasons

6. Market Trends:
   - Trends to watch
   - Implications
   - Action suggestions

7. Final Recommendation:
   - Highlighted top pick
   - Reasoning
   - Immediate next step (actionable)

Include "Start with this opportunity" button that pre-fills a new Niche Validation workflow.
```

---

# DAY 9: Polish, Error Handling & Usage Tracking

## Morning: Error Handling & Edge Cases

### Prompt 9.1 — Global Error Handling

```
Implement comprehensive error handling throughout the frontend.

Files:

1. /components/common/error-boundary.tsx:
   - React error boundary component
   - Catches rendering errors
   - Shows friendly error message
   - "Refresh" and "Go Home" buttons
   - Logs error to console (add TODO for error reporting service)

2. /app/error.tsx:
   - Next.js error page
   - Handles route-level errors
   - Reset functionality

3. /app/not-found.tsx:
   - 404 page
   - Friendly message
   - Navigation suggestions

4. /components/common/api-error.tsx:
   - Component for displaying API errors
   - Different messages for:
     - 400: "Invalid request"
     - 401: "Session expired" with login redirect
     - 403: "Upgrade required" with pricing link
     - 404: "Not found"
     - 429: "Too many requests" with retry timer
     - 500: "Server error" with retry button
   - Retry button where appropriate

5. /lib/api.ts updates:
   - Add response interceptor for error handling
   - Show toast notifications for errors
   - Handle network errors (offline detection)

6. /components/common/offline-indicator.tsx:
   - Banner shown when offline
   - Auto-hides when back online

Ensure all API calls have try-catch with proper error handling.
Add loading states everywhere data is fetched.
```

### Prompt 9.2 — Form Validation & UX Improvements

```
Improve form validation and user experience across all forms.

Updates needed:

1. All forms should use react-hook-form with Zod schemas
2. Real-time validation (validate on blur, show errors inline)
3. Disable submit button when form is invalid
4. Show loading spinner on submit button during API call
5. Clear form errors when user starts typing

Specific improvements:

1. /components/workflows/forms/* :
   - Add tooltips explaining each field
   - Add "info" icons with popovers for complex options
   - URL validation with marketplace detection
   - Budget input with ruble formatting (1000 → 1 000 ₽)
   - Debounced URL validation (check if URL is accessible)

2. /components/auth/* :
   - Password strength meter with requirements checklist
   - Email format validation with common typo detection
   - Show/hide password toggle

3. Add these reusable components:
   - /components/ui/form-field.tsx - wrapper with label, input, error, helper text
   - /components/ui/currency-input.tsx - formatted currency input
   - /components/ui/url-input.tsx - URL input with validation indicator

4. Keyboard navigation:
   - Enter to submit forms
   - Tab order is logical
   - Focus visible states

Add subtle animations for error/success states.
```

## Afternoon: Usage Tracking & Settings

### Prompt 9.3 — Usage Tracking UI

```
Create usage tracking and display throughout the app.

Files:

1. /components/dashboard/usage-widget.tsx:
   - Shows current usage: "X / Y workflows used"
   - Progress bar (changes color as limit approaches)
   - "Upgrade" button when near limit
   - Reset date info

2. /app/(dashboard)/usage/page.tsx:
   - Detailed usage page
   - Current billing period stats
   - Usage history chart (last 30 days)
   - Usage by workflow type breakdown
   - Cost estimates (if we track API costs)

3. /components/dashboard/usage-warning.tsx:
   - Warning banner when:
     - 80% of limit used (yellow)
     - 100% reached (red, blocks new workflows)
   - Upgrade CTA

4. Backend integration:
   - GET /api/usage/summary endpoint
   - Returns: used, limit, reset_date, history

5. /stores/usage-store.ts:
   - Zustand store for usage data
   - Refresh on workflow creation
   - Persist to avoid unnecessary API calls

6. Workflow creation updates:
   - Check usage before showing create form
   - Show warning if near limit
   - Block creation if at limit with upgrade prompt

Add usage check to workflow creation flow.
```

### Prompt 9.4 — Settings Page

```
Create the user settings page.

Files:

1. /app/(dashboard)/settings/page.tsx:
   - Tabbed layout: Profile, Security, Subscription, API Keys

2. /components/settings/profile-settings.tsx:
   - Name update form
   - Email display (read-only with change request)
   - Avatar upload (placeholder for now)
   - Timezone selection

3. /components/settings/security-settings.tsx:
   - Change password form
   - Current password + new password + confirm
   - Password requirements display
   - Last password change date
   - Active sessions list (placeholder)

4. /components/settings/subscription-settings.tsx:
   - Current plan display with features
   - Usage for current period
   - Plan comparison table
   - Upgrade/downgrade buttons (link to pricing/payment - placeholder)
   - Billing history (placeholder)

5. /components/settings/api-keys-settings.tsx:
   - List of API keys
   - Create new key button
   - Key name, created date, last used
   - Reveal/copy key (only shown once on creation)
   - Delete key with confirmation

6. Backend endpoints needed:
   - PUT /api/auth/me (update profile)
   - PUT /api/auth/password (change password)
   - GET /api/keys (list API keys)
   - POST /api/keys (create key)
   - DELETE /api/keys/{id} (delete key)

Add success/error toasts for all actions.
```

---

# DAY 10: Testing, Documentation & Deployment

## Morning: End-to-End Testing

### Prompt 10.1 — Backend API Tests

```
Create comprehensive API tests for all endpoints.

Files:

1. /tests/api/test_workflows.py:
   - test_create_niche_validation_success
   - test_create_niche_validation_invalid_input
   - test_create_workflow_exceeds_limit
   - test_get_workflows_list
   - test_get_workflows_list_with_filters
   - test_get_workflow_detail
   - test_get_workflow_not_found
   - test_get_workflow_unauthorized (other user's workflow)
   - test_delete_workflow_pending
   - test_delete_workflow_running (should fail)
   - test_workflow_status_endpoint

2. /tests/api/test_usage.py:
   - test_get_usage_summary
   - test_usage_increments_on_workflow_create
   - test_usage_limit_enforcement

3. /tests/integration/test_workflow_execution.py:
   - test_niche_validation_full_flow (with mocked APIs)
   - test_competitor_analysis_full_flow
   - test_opportunity_finder_full_flow
   - test_workflow_failure_handling
   - test_workflow_step_tracking

4. /tests/conftest.py updates:
   - Add fixtures for workflows
   - Add mock API response fixtures
   - Add user with various subscription tiers

Use pytest-asyncio and httpx.
Mock all external API calls (Anthropic, OpenAI, Apify).
Aim for >80% code coverage on critical paths.
```

### Prompt 10.2 — Frontend Component Tests

```
Create tests for critical frontend components.

Testing setup:
- Vitest for test runner
- React Testing Library for component tests
- MSW (Mock Service Worker) for API mocking

Files:

1. /tests/components/auth.test.tsx:
   - Login form renders correctly
   - Login form validates inputs
   - Login form submits and redirects on success
   - Login form shows error on failure
   - Register form validates password strength
   - Register form shows success message

2. /tests/components/workflows.test.tsx:
   - Workflow list renders items
   - Workflow list shows loading state
   - Workflow list shows empty state
   - Workflow card shows correct status
   - Workflow creation form validates
   - Workflow creation submits correctly

3. /tests/components/results.test.tsx:
   - Niche validation results render all sections
   - Competitor analysis results render correctly
   - Opportunity finder results show rankings

4. /tests/hooks/use-auth.test.tsx:
   - useAuth returns correct state
   - login updates state correctly
   - logout clears state

5. /tests/mocks/handlers.ts:
   - MSW handlers for all API endpoints
   - Success and error scenarios

Focus on user interactions and critical paths.
Add snapshot tests for complex result displays.
```

## Afternoon: Documentation & Deployment

### Prompt 10.3 — API Documentation

```
Create comprehensive API documentation.

Files:

1. /docs/api.md:
   - Authentication section:
     - How to register
     - How to login
     - How to use JWT tokens
     - Token refresh flow
   
   - Workflows section:
     - Available workflow types
     - Request/response examples for each
     - Status codes and error responses
   
   - Usage section:
     - How usage is tracked
     - Limits by subscription tier
   
   - Rate limiting info

2. Update FastAPI with OpenAPI enhancements:
   - Add descriptions to all endpoints
   - Add example requests/responses
   - Add tags for grouping
   - Add security schemes

3. /docs/development.md:
   - Local setup instructions
   - Environment variables reference
   - Database migrations guide
   - Running tests
   - Code style guidelines

4. /docs/deployment.md:
   - Production environment setup
   - Docker deployment guide
   - Environment variables for production
   - Health checks
   - Monitoring setup (TODO)

5. README.md updates:
   - Project overview
   - Quick start
   - Links to documentation
   - Contributing guidelines
```

### Prompt 10.4 — Production Deployment Setup

```
Create production-ready deployment configuration.

Files:

1. /docker-compose.prod.yml:
   - Production-optimized services
   - No volume mounts for code
   - Health checks for all services
   - Restart policies
   - Resource limits
   - Production environment variables

2. /backend/Dockerfile.prod:
   - Multi-stage build
   - Non-root user
   - Optimized for size
   - Health check command

3. /frontend/Dockerfile.prod:
   - Multi-stage build (build + nginx)
   - Optimized static serving
   - Security headers in nginx

4. /nginx/nginx.conf:
   - Reverse proxy to API
   - Static file serving for frontend
   - Gzip compression
   - SSL termination (placeholder for certs)
   - Rate limiting
   - Security headers

5. /.github/workflows/deploy.yml (or equivalent CI/CD):
   - Build and test on PR
   - Build Docker images on main
   - Push to container registry
   - Deploy to server (placeholder)

6. /scripts/deploy.sh:
   - Pull latest images
   - Run migrations
   - Start/restart services
   - Health check verification

7. Environment files:
   - .env.production.example
   - List all required production variables

Add comments explaining each configuration choice.
```

### Prompt 10.5 — Final Checklist & Handoff

```
Create a final checklist and handoff documentation.

Files:

1. /docs/launch-checklist.md:
   
   ## Pre-Launch Checklist
   
   ### Security
   - [ ] All secrets in environment variables
   - [ ] JWT secret is strong and unique
   - [ ] API keys are not exposed in frontend
   - [ ] CORS configured correctly
   - [ ] Rate limiting enabled
   - [ ] SQL injection prevention verified
   - [ ] XSS prevention verified
   
   ### Performance
   - [ ] Database indexes created
   - [ ] API responses are fast (<500ms)
   - [ ] Frontend bundle size optimized
   - [ ] Images optimized
   - [ ] Caching configured
   
   ### Reliability
   - [ ] Error handling covers all cases
   - [ ] Failed workflows can be retried
   - [ ] Database backups configured
   - [ ] Monitoring alerts set up
   
   ### User Experience
   - [ ] All forms validate properly
   - [ ] Loading states everywhere
   - [ ] Error messages are helpful
   - [ ] Mobile responsive
   
   ### Testing
   - [ ] All critical paths tested
   - [ ] API tests passing
   - [ ] Frontend tests passing
   - [ ] Manual testing completed

2. /docs/known-issues.md:
   - List any known issues or TODOs
   - Workarounds if applicable

3. /docs/future-improvements.md:
   - Features for next phase
   - Technical debt to address
   - Performance optimizations

4. /docs/troubleshooting.md:
   - Common issues and solutions
   - How to check logs
   - How to restart services
   - Contact for support

Create clear, actionable documentation for handoff.
```

---
