# 📋 10-DAY TECHNICAL ROADMAP

## **Project: Uzbek Academic Paper Assistant MVP**
**Goal:** Launch testable MVP with 20 university students  
**Timeline:** 10 days  
**Stack:** Next.js + Supabase + GPT-4o/mini API

---

## **DAY 1: ARCHITECTURE & SETUP**

### **Objectives:**
- [ ] Finalize system architecture
- [ ] Set up development environment
- [ ] Create project structure

### **Deliverables:**
- System architecture diagram
- Database schema design
- API endpoint specifications
- Environment setup complete

### **Technical Decisions:**
| Component | Choice | Reason |
|-----------|--------|--------|
| Frontend | Next.js 14 | Fast development |
| Backend | Next.js API routes | Simplified stack |
| Database | Supabase (PostgreSQL) | Easy auth + storage |
| File Storage | Supabase Storage | PDF handling |
| AI API | OpenAI (GPT-4o/mini) | Best for academic writing |
| PDF Parsing | pdf-parse / pdf.js | Extract text |

### **Database Schema:**
```sql
-- Users table (handled by Supabase Auth)

-- Papers table
CREATE TABLE papers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users NOT NULL,
  title TEXT NOT NULL,
  original_file_url TEXT NOT NULL,
  extracted_text TEXT NOT NULL,
  page_count INTEGER,
  language TEXT DEFAULT 'uzbek',
  status TEXT DEFAULT 'uploaded', -- uploaded, processing, completed
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Feedback rounds table
CREATE TABLE feedback_rounds (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  paper_id UUID REFERENCES papers(id) ON DELETE CASCADE,
  round_number INTEGER NOT NULL, -- 1, 2, 3
  model_used TEXT NOT NULL, -- gpt-4o, gpt-4o-mini
  content_feedback TEXT,
  structure_feedback TEXT,
  revised_text TEXT,
  changes_summary TEXT,
  tokens_used INTEGER,
  cost_usd DECIMAL(10,4),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- User subscriptions
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users NOT NULL,
  plan_type TEXT DEFAULT 'free', -- free, paid
  papers_remaining INTEGER DEFAULT 2,
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Usage tracking
CREATE TABLE usage_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users,
  paper_id UUID REFERENCES papers,
  action TEXT NOT NULL, -- upload, feedback_r1, feedback_r2, etc
  tokens_used INTEGER,
  cost_usd DECIMAL(10,4),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### **API Endpoints Specification:**
```
POST /api/papers/upload         - Upload paper (PDF/DOCX)
GET  /api/papers/:id            - Get paper details
POST /api/papers/:id/feedback   - Generate feedback (round 1-3)
GET  /api/papers/:id/comparison - Get before/after view
POST /api/papers/:id/export     - Export formatted paper
GET  /api/user/usage            - Get user's remaining papers
POST /api/payment/subscribe     - Handle subscription
```

---

## **DAY 2: PDF PROCESSING PIPELINE**

### **Objectives:**
- [ ] Build PDF upload functionality
- [ ] Extract text from PDFs
- [ ] Store papers in database

### **Deliverables:**
- Working PDF upload endpoint
- Text extraction pipeline
- Storage in Supabase

### **Technical Implementation:**

**File Upload Flow:**
```
User uploads PDF 
→ Validate (size, format)
→ Store in Supabase Storage
→ Extract text (pdf-parse)
→ Clean text (remove headers/footers/page numbers)
→ Store in database
→ Return paper ID
```

**Text Cleaning Rules:**
- Remove page numbers
- Remove headers/footers
- Normalize whitespace
- Detect language (Uzbek/Russian/English)
- Count actual content pages

### **Testing Checklist:**
- [ ] Upload 5MB PDF - works
- [ ] Upload 50MB PDF - handles gracefully
- [ ] Upload corrupted PDF - shows error
- [ ] Extract Uzbek text - preserves formatting
- [ ] Extract Russian text - preserves Cyrillic

---

## **DAY 3: UZBEK PAPER STRUCTURE TEMPLATE**

### **Objectives:**
- [ ] Document official Uzbek university paper structure
- [ ] Create structure validation rules
- [ ] Build structure detection algorithm

### **Deliverables:**
- Uzbek paper structure template document
- Structure validation JSON schema
- Detection algorithm

### **Official Structure (Document This):**
```markdown
# Standard Uzbek Research Paper Structure

## Required Sections (in order):
1. Title Page
   - Title (Uzbek + English)
   - Author info
   - University
   - Year

2. Annotatsiya (Abstract in Uzbek)
   - 150-250 words
   - Kalit so'zlar (Keywords): 5-7 words

3. Annotation (Abstract in Russian/English)
   - 150-250 words
   - Keywords: 5-7 words

4. Kirish (Introduction)
   - Research problem
   - Relevance
   - Goals and objectives

5. Adabiyotlar tahlili (Literature Review)
   - Current state of research
   - Gaps identified

6. Tadqiqot metodologiyasi (Methodology)
   - Research methods
   - Data collection

7. Natijalar va muhokama (Results and Discussion)
   - Findings
   - Analysis

8. Xulosa (Conclusion)
   - Summary
   - Recommendations

9. Foydalanilgan adabiyotlar (References)
   - Format: [Author. Title. Publisher, Year. Pages]
```

### **Structure Detection Algorithm:**
```javascript
// Pseudo-code
function detectStructure(paperText) {
  const sections = [
    { name: 'Annotatsiya', keywords: ['annotatsiya', 'abstract', 'kalit so\'zlar'] },
    { name: 'Kirish', keywords: ['kirish', 'introduction', 'dolzarbligi'] },
    { name: 'Adabiyotlar tahlili', keywords: ['adabiyotlar', 'literature', 'tahlili'] },
    // ... etc
  ];
  
  return {
    detected: [...],
    missing: [...],
    order_correct: boolean,
    compliance_score: 0-100
  };
}
```

---

## **DAY 4: ROUND 1 FEEDBACK - CONTENT ANALYSIS**

### **Objectives:**
- [ ] Build GPT-4o prompt for content feedback
- [ ] Implement API call with token tracking
- [ ] Store feedback in database

### **Deliverables:**
- Content analysis prompt (optimized)
- Working API endpoint
- Token/cost tracking

### **GPT-4o Content Feedback Prompt:**
```markdown
You are an expert Uzbek academic paper reviewer specializing in university-level research papers.

**Task:** Analyze this research paper and provide detailed feedback on content quality.

**Paper Metadata:**
- Title: {title}
- Length: {page_count} pages
- Language: {detected_language}

**Paper Content:**
{paper_text}

**Provide feedback in JSON format:**
{
  "overall_assessment": {
    "score": 0-100,
    "strengths": ["strength 1", "strength 2"],
    "weaknesses": ["weakness 1", "weakness 2"]
  },
  "section_feedback": {
    "introduction": {
      "score": 0-100,
      "comments": "Detailed feedback",
      "suggestions": ["suggestion 1", "suggestion 2"]
    },
    "literature_review": { ... },
    "methodology": { ... },
    "results": { ... },
    "conclusion": { ... }
  },
  "academic_quality": {
    "research_question_clarity": 0-100,
    "argument_coherence": 0-100,
    "evidence_quality": 0-100,
    "critical_analysis": 0-100
  },
  "language_quality": {
    "grammar": 0-100,
    "academic_tone": 0-100,
    "terminology_accuracy": 0-100
  },
  "specific_issues": [
    {
      "location": "Page 5, paragraph 3",
      "issue": "Description",
      "severity": "high|medium|low",
      "suggestion": "How to fix"
    }
  ],
  "priority_improvements": ["Top 5 things to fix"]
}

**Output in Uzbek language for Uzbek papers, Russian for Russian papers.**
```

### **API Implementation:**
```typescript
// /api/papers/:id/feedback
export async function POST(req: Request) {
  // 1. Get paper from DB
  // 2. Check user has papers remaining
  // 3. Determine round number (1, 2, or 3)
  // 4. Select model (GPT-4o for R1, mini for R2-3)
  // 5. Call OpenAI API with prompt
  // 6. Track tokens & cost
  // 7. Store feedback in DB
  // 8. Decrement user's paper count
  // 9. Return feedback
}
```

### **Cost Tracking:**
```typescript
const PRICING = {
  'gpt-4o': { input: 2.50, output: 10.00 }, // per 1M tokens
  'gpt-4o-mini': { input: 0.150, output: 0.600 }
};

function calculateCost(model: string, inputTokens: number, outputTokens: number) {
  const prices = PRICING[model];
  return (inputTokens / 1_000_000 * prices.input) + 
         (outputTokens / 1_000_000 * prices.output);
}
```

---

## **DAY 5: ROUND 1 FEEDBACK - STRUCTURE VALIDATION**

### **Objectives:**
- [ ] Build structure compliance checker
- [ ] Generate structure feedback
- [ ] Combine with content feedback

### **Deliverables:**
- Structure validation algorithm
- Structure feedback prompt
- Combined feedback UI

### **Structure Feedback Prompt:**
```markdown
You are an expert in Uzbek university research paper formatting standards.

**Official Structure Required:**
{uzbek_structure_template}

**Detected Structure in Paper:**
{detected_sections}

**Task:** Analyze structure compliance and provide detailed formatting feedback.

**Output JSON:**
{
  "compliance_score": 0-100,
  "structure_analysis": {
    "correct_sections": ["section 1", "section 2"],
    "missing_sections": ["section X"],
    "incorrect_order": ["section Y should come before Z"],
    "formatting_issues": [
      {
        "section": "Annotatsiya",
        "issue": "Should be 150-250 words, found 89 words",
        "fix": "Expand abstract with more detail on methodology and results"
      }
    ]
  },
  "title_page": {
    "has_uzbek_title": boolean,
    "has_english_title": boolean,
    "has_author_info": boolean,
    "issues": []
  },
  "abstract_analysis": {
    "uzbek_abstract_word_count": number,
    "has_keywords": boolean,
    "keyword_count": number,
    "issues": []
  },
  "reference_format": {
    "total_references": number,
    "format_compliance": 0-100,
    "common_errors": []
  },
  "priority_fixes": ["Fix 1", "Fix 2", "Fix 3"]
}

**Output in Uzbek language.**
```

### **Combined Feedback Display:**
```
┌─────────────────────────────────────┐
│ PAPER ANALYSIS COMPLETE             │
├─────────────────────────────────────┤
│ Content Quality:        78/100  🟡  │
│ Structure Compliance:   92/100  🟢  │
│ Overall Score:          85/100      │
└─────────────────────────────────────┘

📝 CONTENT FEEDBACK
  ✅ Strengths: Clear research question, Strong methodology
  ⚠️  Needs Work: Weak literature review, Limited data analysis

📐 STRUCTURE FEEDBACK
  ✅ All required sections present
  ⚠️  Abstract too short (89 words, need 150-250)
  ⚠️  Missing English translation of title

🎯 TOP PRIORITY FIXES:
  1. Expand abstract to 200+ words
  2. Add 5-10 more literature sources
  3. Strengthen results section with data visualization
```

---

## **DAY 6: BEFORE/AFTER COMPARISON VIEW**

### **Objectives:**
- [ ] Build diff visualization component
- [ ] Show changes side-by-side
- [ ] Highlight improvements

### **Deliverables:**
- Comparison UI component
- Change tracking system
- Export functionality

### **UI Component Structure:**
```tsx
// components/ComparisonView.tsx

<div className="grid grid-cols-2 gap-4">
  {/* Original Version */}
  <div className="border-r pr-4">
    <h3>Original</h3>
    <div className="paper-content">
      {sections.map(section => (
        <Section 
          content={section.original}
          highlights={section.issues}
        />
      ))}
    </div>
  </div>

  {/* Improved Version */}
  <div className="pl-4">
    <h3>Suggested Improvements</h3>
    <div className="paper-content">
      {sections.map(section => (
        <Section 
          content={section.revised}
          highlights={section.changes}
          showDiff={true}
        />
      ))}
    </div>
  </div>
</div>

{/* Change Summary */}
<div className="mt-4 border-t pt-4">
  <h4>Summary of Changes</h4>
  <ul>
    <li>✏️ 12 grammar corrections</li>
    <li>📝 3 sections expanded</li>
    <li>📚 5 citation format fixes</li>
    <li>🔄 Structure reordered</li>
  </ul>
</div>
```

### **Diff Algorithm:**
- Use `diff-match-patch` library for text comparison
- Highlight additions in green
- Highlight deletions in red
- Show inline suggestions

### **Export Options:**
- Export as DOCX with track changes
- Export as PDF with annotations
- Export feedback report as PDF

---

## **DAY 7: ROUNDS 2-3 REFINEMENT**

### **Objectives:**
- [ ] Build iterative refinement flow
- [ ] Use GPT-4o-mini for cost efficiency
- [ ] Track improvements across rounds

### **Deliverables:**
- Refinement prompt
- Multi-round tracking
- Progress visualization

### **Round 2-3 Prompt (GPT-4o-mini):**
```markdown
You are refining an Uzbek academic paper based on previous feedback.

**Original Paper:**
{original_text}

**Round 1 Feedback:**
{round_1_feedback}

**User's Revised Version:**
{user_revised_text}

**Task:** Provide focused feedback on:
1. Whether previous issues were addressed
2. Any new issues introduced
3. Remaining improvements needed

**Keep feedback CONCISE and SPECIFIC. Focus only on:**
- Unresolved issues from Round 1
- New problems in revised version
- Final polish suggestions

**Output JSON:**
{
  "progress_score": 0-100,
  "issues_resolved": ["issue 1", "issue 2"],
  "issues_remaining": [
    {
      "issue": "Description",
      "location": "Page X",
      "priority": "high|medium|low"
    }
  ],
  "new_suggestions": ["suggestion 1", "suggestion 2"],
  "ready_for_submission": boolean,
  "final_notes": "Brief summary"
}
```

### **Progress Tracking:**
```
Round 1 → Round 2 → Round 3
  78       →   85    →   92
  
✅ Resolved Issues (5):
  • Abstract length fixed
  • Added 8 more references
  • Improved methodology section
  • Fixed citation format
  • Strengthened conclusion

⚠️ Remaining Issues (2):
  • Data visualization needed in results
  • Minor grammar corrections

🎯 Ready for submission: YES
```

---

## **DAY 8: CITATION FORMATTER**

### **Objectives:**
- [ ] Build citation parser
- [ ] Auto-format to Uzbek standard
- [ ] Validate bibliography

### **Deliverables:**
- Citation detection algorithm
- Auto-formatter
- Validation rules

### **Uzbek Citation Standard:**
```
Format: Author. Title. – Publisher: City, Year. – Pages.

Examples:
  Book:
    Karimov I.A. O'zbekiston XXI asr bo'sag'asida. – T.: O'zbekiston, 1997. – 320 b.
  
  Article:
    Ergashev B. Milliy g'oya asoslari // Tafakkur. – 2020. – № 2. – B. 15-23.
  
  Online:
    Rahmonov S. Digital iqtisodiyot. [Electronic resource] – URL: https://example.uz (20.01.2024).
```

### **Citation Parser Algorithm:**
```javascript
function parseCitation(rawText) {
  // 1. Detect citation type (book, article, online)
  // 2. Extract components (author, title, year, etc)
  // 3. Validate format
  // 4. Return structured data
  
  return {
    type: 'book' | 'article' | 'online',
    author: string,
    title: string,
    publisher: string,
    year: number,
    pages: string,
    is_valid: boolean,
    errors: []
  };
}
```

### **Auto-Formatter:**
```typescript
function formatCitation(citation: ParsedCitation): string {
  switch(citation.type) {
    case 'book':
      return `${citation.author}. ${citation.title}. – ${citation.publisher}: ${citation.city}, ${citation.year}. – ${citation.pages} b.`;
    case 'article':
      return `${citation.author}. ${citation.title} // ${citation.journal}. – ${citation.year}. – № ${citation.issue}. – B. ${citation.pages}.`;
    case 'online':
      return `${citation.author}. ${citation.title}. [Electron resurs] – URL: ${citation.url} (${citation.access_date}).`;
  }
}
```

### **Bibliography Validator:**
- Check all in-text citations have references
- Check all references are cited in text
- Alphabetically sort (Uzbek alphabetical order)
- Number references consistently

---

## **DAY 9: USER DASHBOARD & USAGE TRACKING**

### **Objectives:**
- [ ] Build user dashboard
- [ ] Show paper history
- [ ] Display usage/limits

### **Deliverables:**
- Dashboard UI
- Usage tracking system
- Subscription management

### **Dashboard Features:**

```tsx
// Dashboard Layout
<div className="dashboard">
  {/* Usage Overview */}
  <div className="usage-card">
    <h3>Your Subscription</h3>
    <p>Plan: Premium ($10/month)</p>
    <p>Papers Remaining: 1 / 2</p>
    <p>Resets: March 1, 2025</p>
    <Button>Buy Extra Paper ($5)</Button>
  </div>

  {/* Paper History */}
  <div className="papers-list">
    <h3>Your Papers</h3>
    {papers.map(paper => (
      <PaperCard
        title={paper.title}
        status={paper.status} // Processing, Completed
        rounds_completed={paper.rounds_completed}
        score={paper.latest_score}
        last_updated={paper.updated_at}
        actions={
          <div>
            <Button>View Feedback</Button>
            <Button>Continue Editing</Button>
            <Button>Download</Button>
          </div>
        }
      />
    ))}
  </div>

  {/* Quick Stats */}
  <div className="stats">
    <Stat label="Papers Reviewed" value={15} />
    <Stat label="Avg Score Improvement" value="+18 points" />
    <Stat label="Success Rate" value="94%" />
  </div>
</div>
```

### **Usage Tracking System:**
```sql
-- Track every action for analytics
INSERT INTO usage_logs (user_id, action, paper_id, tokens_used, cost_usd)
VALUES ($1, $2, $3, $4, $5);

-- Check if user can submit paper
SELECT papers_remaining FROM subscriptions 
WHERE user_id = $1 AND expires_at > NOW();

-- Decrement usage
UPDATE subscriptions 
SET papers_remaining = papers_remaining - 1 
WHERE user_id = $1;
```

---

## **DAY 10: TESTING & DEPLOYMENT**

### **Objectives:**
- [ ] End-to-end testing with real papers
- [ ] Deploy to production
- [ ] Onboard 20 test users

### **Deliverables:**
- Tested MVP on production
- User onboarding flow
- Feedback collection system

### **Testing Checklist:**

**Functional Testing:**
- [ ] Upload 5 different papers (varying lengths)
- [ ] Complete 3 full rounds on 1 paper
- [ ] Test all citation formats
- [ ] Export papers in all formats
- [ ] Test subscription limits
- [ ] Test payment flow (Stripe test mode)

**Performance Testing:**
- [ ] Upload 10 papers simultaneously
- [ ] Measure API response times
- [ ] Check database query performance
- [ ] Monitor token usage accuracy

**User Experience Testing:**
- [ ] Complete user flow without instructions
- [ ] Test on mobile devices
- [ ] Test in Uzbek/Russian languages
- [ ] Measure time to first feedback

### **Deployment Steps:**

```bash
# 1. Environment setup
vercel env add OPENAI_API_KEY
vercel env add SUPABASE_URL
vercel env add SUPABASE_ANON_KEY
vercel env add STRIPE_SECRET_KEY

# 2. Deploy to production
vercel --prod

# 3. Run database migrations
supabase db push --db-url $PRODUCTION_DB_URL

# 4. Seed initial data
npm run seed:production
```

### **User Onboarding (20 Students):**

**Onboarding Email Template:**
```
Subject: You're invited to test our Academic Paper Assistant! 🎓

Hi [Name],

You're one of 20 students selected to test our new AI-powered paper review tool!

What you get:
✅ FREE access for 1 month
✅ Review up to 5 papers
✅ Get detailed feedback in minutes
✅ Improve your academic writing

Getting started:
1. Create account: https://ilmai.uz/signup?code=TEST20
2. Upload your paper (PDF or DOCX)
3. Get instant feedback

We need your honest feedback to improve!

Login: [Your email]
Access code: TEST20

Questions? Reply to this email.

Good luck with your research! 📚
IlmAI Team
```

### **Feedback Collection:**
```tsx
// After each paper review, show survey
<FeedbackModal>
  <h3>How was your experience?</h3>
  
  <Rating question="Feedback quality (1-5)" />
  <Rating question="Ease of use (1-5)" />
  <Rating question="Speed (1-5)" />
  
  <TextArea 
    label="What did you like most?" 
    required 
  />
  
  <TextArea 
    label="What needs improvement?" 
    required 
  />
  
  <Checkbox 
    label="Would you pay $10/month for this?" 
  />
  
  <Button>Submit Feedback</Button>
</FeedbackModal>
```

### **Success Metrics (Track Daily):**
- Papers uploaded: Target 40+ papers in 10 days
- Completion rate: Target 80% complete all 3 rounds
- User satisfaction: Target 4.0+ / 5.0 average
- Technical issues: Target <5% error rate
- Cost per paper: Target <$0.20 average

---

# 💻 10-DAY CURSOR PROMPTS FOR JUNIOR DEVS

## **DAY 1: PROJECT SETUP**

```
You are building "IlmAI" - an AI-powered academic paper review tool for Uzbek university students.

TECH STACK:
- Next.js 14 (App Router) with TypeScript
- Supabase (PostgreSQL + Auth + Storage)
- OpenAI API (GPT-4o and GPT-4o-mini)
- Tailwind CSS
- Shadcn/ui components

TASK: Set up the complete project foundation

1. Create new Next.js 14 project:
   - TypeScript enabled
   - App Router (not Pages Router)
   - Tailwind CSS
   - ESLint

2. Install core dependencies:
   npm install @supabase/supabase-js @supabase/auth-helpers-nextjs
   npm install openai
   npm install pdf-parse
   npm install @radix-ui/react-* (for shadcn/ui)
   npm install lucide-react (icons)
   npm install date-fns (date formatting)
   npm install react-hot-toast (notifications)

3. Project structure:
   /app
     /api
       /papers
         /upload/route.ts
         /[id]/feedback/route.ts
         /[id]/route.ts
       /user
         /usage/route.ts
     /(auth)
       /login/page.tsx
       /signup/page.tsx
     /dashboard/page.tsx
     /papers/[id]/page.tsx
     layout.tsx
     page.tsx
   /components
     /ui (shadcn components)
     /papers
       PaperCard.tsx
       PaperUpload.tsx
       FeedbackView.tsx
       ComparisonView.tsx
     /dashboard
       UsageCard.tsx
       PapersList.tsx
   /lib
     supabase.ts
     openai.ts
     utils.ts
   /types
     paper.ts
     feedback.ts
     user.ts

4. Set up environment variables (.env.local):
   NEXT_PUBLIC_SUPABASE_URL=
   NEXT_PUBLIC_SUPABASE_ANON_KEY=
   SUPABASE_SERVICE_KEY=
   OPENAI_API_KEY=
   NEXT_PUBLIC_APP_URL=http://localhost:3000

5. Create Supabase client utilities in /lib/supabase.ts:
   - Client-side Supabase client
   - Server-side Supabase client
   - Auth helpers

6. Create OpenAI client in /lib/openai.ts:
   - Initialize OpenAI with API key
   - Helper function for chat completions
   - Token counting utility

7. Set up Tailwind config for Uzbek language fonts:
   - Add Inter for English
   - Add system fonts for Cyrillic

8. Initialize shadcn/ui:
   npx shadcn-ui@latest init
   - Install: button, card, input, textarea, dialog, dropdown-menu

9. Create basic TypeScript types in /types/paper.ts:
   - Paper interface
   - FeedbackRound interface
   - UserSubscription interface

10. Deploy skeleton to Vercel:
    - Connect GitHub repo
    - Set environment variables
    - Verify deployment works

DELIVERABLES:
- Fully configured Next.js project
- All dependencies installed
- Clean folder structure
- Type definitions created
- Deployed skeleton on Vercel
- Share deployment URL

IMPORTANT:
- Use "use client" directive only when necessary
- Prefer Server Components by default
- Use async/await for all API calls
- Add proper error handling everywhere
```

---

## **DAY 2: SUPABASE DATABASE SETUP**

```
TASK: Set up complete Supabase database schema and storage

1. In Supabase Dashboard SQL Editor, create database schema:

-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Papers table
CREATE TABLE papers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) NOT NULL,
  title TEXT NOT NULL,
  original_file_url TEXT NOT NULL,
  extracted_text TEXT NOT NULL,
  page_count INTEGER,
  language TEXT DEFAULT 'uzbek' CHECK (language IN ('uzbek', 'russian', 'english')),
  status TEXT DEFAULT 'uploaded' CHECK (status IN ('uploaded', 'processing', 'completed', 'failed')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Feedback rounds table
CREATE TABLE feedback_rounds (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  paper_id UUID REFERENCES papers(id) ON DELETE CASCADE,
  round_number INTEGER NOT NULL CHECK (round_number BETWEEN 1 AND 3),
  model_used TEXT NOT NULL,
  content_feedback JSONB,
  structure_feedback JSONB,
  revised_text TEXT,
  changes_summary TEXT,
  tokens_used INTEGER,
  cost_usd DECIMAL(10,4),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Subscriptions table
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) NOT NULL UNIQUE,
  plan_type TEXT DEFAULT 'free' CHECK (plan_type IN ('free', 'paid')),
  papers_remaining INTEGER DEFAULT 2,
  total_papers_allowed INTEGER DEFAULT 2,
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Usage logs table
CREATE TABLE usage_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id),
  paper_id UUID REFERENCES papers(id),
  action TEXT NOT NULL,
  tokens_used INTEGER,
  cost_usd DECIMAL(10,4),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create indexes for performance
CREATE INDEX idx_papers_user_id ON papers(user_id);
CREATE INDEX idx_papers_status ON papers(status);
CREATE INDEX idx_feedback_paper_id ON feedback_rounds(paper_id);
CREATE INDEX idx_usage_logs_user_id ON usage_logs(user_id);
CREATE INDEX idx_usage_logs_created_at ON usage_logs(created_at);

-- Enable Row Level Security
ALTER TABLE papers ENABLE ROW LEVEL SECURITY;
ALTER TABLE feedback_rounds ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE usage_logs ENABLE ROW LEVEL SECURITY;

-- RLS Policies for papers
CREATE POLICY "Users can view own papers"
  ON papers FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own papers"
  ON papers FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own papers"
  ON papers FOR UPDATE
  USING (auth.uid() = user_id);

-- RLS Policies for feedback_rounds
CREATE POLICY "Users can view feedback for own papers"
  ON feedback_rounds FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM papers 
      WHERE papers.id = feedback_rounds.paper_id 
      AND papers.user_id = auth.uid()
    )
  );

-- RLS Policies for subscriptions
CREATE POLICY "Users can view own subscription"
  ON subscriptions FOR SELECT
  USING (auth.uid() = user_id);

-- Function to auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Triggers for updated_at
CREATE TRIGGER update_papers_updated_at
  BEFORE UPDATE ON papers
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_subscriptions_updated_at
  BEFORE UPDATE ON subscriptions
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

2. Set up Supabase Storage:
   - Create bucket: "papers"
   - Set bucket to private
   - Create RLS policy: Users can only access own files

3. Update /lib/supabase.ts with helper functions:
   - getUserPapers(userId)
   - getPaperById(paperId)
   - createPaper(paperData)
   - updatePaperStatus(paperId, status)
   - getUserSubscription(userId)
   - decrementPaperCount(userId)

4. Create seed data for testing:
   - Insert test user subscription
   - Insert 2 sample papers
   - Insert sample feedback rounds

5. Test all database operations:
   - Create paper
   - Fetch papers
   - Update paper
   - Delete paper
   - Check RLS policies work

DELIVERABLES:
- Complete database schema created
- RLS policies working
- Storage bucket configured
- Helper functions in lib/supabase.ts
- Seed data inserted
- All operations tested

TESTING CHECKLIST:
- [ ] Can create paper as authenticated user
- [ ] Cannot access other user's papers
- [ ] Subscription decrements correctly
- [ ] Updated_at triggers work
- [ ] Can upload file to storage
```

---

## **DAY 3: PDF UPLOAD & TEXT EXTRACTION**

```
TASK: Build complete PDF upload and text extraction pipeline

1. Install PDF processing libraries:
   npm install pdf-parse
   npm install mammoth (for DOCX support)

2. Create PDF processing utility in /lib/pdf-processor.ts:

import pdfParse from 'pdf-parse';
import mammoth from 'mammoth';

interface ExtractedContent {
  text: string;
  pageCount: number;
  language: 'uzbek' | 'russian' | 'english';
}

export async function extractTextFromPDF(buffer: Buffer): Promise<ExtractedContent> {
  // 1. Parse PDF
  const data = await pdfParse(buffer);
  
  // 2. Clean text
  const cleanedText = cleanText(data.text);
  
  // 3. Detect language
  const language = detectLanguage(cleanedText);
  
  // 4. Count pages
  const pageCount = data.numpages;
  
  return {
    text: cleanedText,
    pageCount,
    language
  };
}

function cleanText(rawText: string): string {
  // Remove page numbers
  let cleaned = rawText.replace(/^\d+\s*$/gm, '');
  
  // Remove excessive whitespace
  cleaned = cleaned.replace(/\s+/g, ' ');
  
  // Remove common headers/footers patterns
  cleaned = cleaned.replace(/Page \d+ of \d+/gi, '');
  
  // Normalize line breaks
  cleaned = cleaned.replace(/\r\n/g, '\n');
  
  return cleaned.trim();
}

function detectLanguage(text: string): 'uzbek' | 'russian' | 'english' {
  // Count Cyrillic vs Latin characters
  const cyrillicCount = (text.match(/[\u0400-\u04FF]/g) || []).length;
  const latinCount = (text.match(/[a-zA-Z]/g) || []).length;
  
  // Check for Uzbek-specific patterns
  const uzbekPatterns = ['maqsad', 'tadqiqot', 'natija', 'xulosa', 'kirish'];
  const hasUzbekWords = uzbekPatterns.some(word => 
    text.toLowerCase().includes(word)
  );
  
  if (hasUzbekWords) return 'uzbek';
  if (cyrillicCount > latinCount) return 'russian';
  return 'english';
}

export async function extractTextFromDOCX(buffer: Buffer): Promise<ExtractedContent> {
  const result = await mammoth.extractRawText({ buffer });
  const cleanedText = cleanText(result.value);
  const language = detectLanguage(cleanedText);
  
  // Estimate page count (500 words per page)
  const wordCount = cleanedText.split(/\s+/).length;
  const pageCount = Math.ceil(wordCount / 500);
  
  return {
    text: cleanedText,
    pageCount,
    language
  };
}

3. Create file upload API route in /app/api/papers/upload/route.ts:

import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { extractTextFromPDF, extractTextFromDOCX } from '@/lib/pdf-processor';

export async function POST(request: Request) {
  try {
    // 1. Get authenticated user
    const supabase = createRouteHandlerClient({ cookies });
    const { data: { user }, error: authError } = await supabase.auth.getUser();
    
    if (authError || !user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    // 2. Check user has papers remaining
    const { data: subscription } = await supabase
      .from('subscriptions')
      .select('papers_remaining')
      .eq('user_id', user.id)
      .single();
    
    if (!subscription || subscription.papers_remaining <= 0) {
      return NextResponse.json(
        { error: 'No papers remaining. Please upgrade.' },
        { status: 403 }
      );
    }

    // 3. Get file from form data
    const formData = await request.formData();
    const file = formData.get('file') as File;
    
    if (!file) {
      return NextResponse.json({ error: 'No file provided' }, { status: 400 });
    }

    // 4. Validate file
    const maxSize = 50 * 1024 * 1024; // 50MB
    if (file.size > maxSize) {
      return NextResponse.json(
        { error: 'File too large. Maximum 50MB.' },
        { status: 400 }
      );
    }

    const allowedTypes = ['application/pdf', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'];
    if (!allowedTypes.includes(file.type)) {
      return NextResponse.json(
        { error: 'Invalid file type. Only PDF and DOCX allowed.' },
        { status: 400 }
      );
    }

    // 5. Upload file to Supabase Storage
    const fileBuffer = Buffer.from(await file.arrayBuffer());
    const fileName = `${user.id}/${Date.now()}_${file.name}`;
    
    const { data: uploadData, error: uploadError } = await supabase.storage
      .from('papers')
      .upload(fileName, fileBuffer, {
        contentType: file.type,
        upsert: false
      });

    if (uploadError) {
      throw new Error(`Upload failed: ${uploadError.message}`);
    }

    // 6. Get public URL
    const { data: { publicUrl } } = supabase.storage
      .from('papers')
      .getPublicUrl(fileName);

    // 7. Extract text from PDF/DOCX
    let extracted;
    if (file.type === 'application/pdf') {
      extracted = await extractTextFromPDF(fileBuffer);
    } else {
      extracted = await extractTextFromDOCX(fileBuffer);
    }

    // 8. Create paper record in database
    const { data: paper, error: dbError } = await supabase
      .from('papers')
      .insert({
        user_id: user.id,
        title: file.name.replace(/\.(pdf|docx)$/i, ''),
        original_file_url: publicUrl,
        extracted_text: extracted.text,
        page_count: extracted.pageCount,
        language: extracted.language,
        status: 'uploaded'
      })
      .select()
      .single();

    if (dbError) {
      throw new Error(`Database error: ${dbError.message}`);
    }

    // 9. Log usage
    await supabase.from('usage_logs').insert({
      user_id: user.id,
      paper_id: paper.id,
      action: 'upload',
      metadata: {
        file_name: file.name,
        file_size: file.size,
        page_count: extracted.pageCount
      }
    });

    // 10. Return paper details
    return NextResponse.json({
      success: true,
      paper: {
        id: paper.id,
        title: paper.title,
        pageCount: paper.page_count,
        language: paper.language,
        status: paper.status
      }
    });

  } catch (error) {
    console.error('Upload error:', error);
    return NextResponse.json(
      { error: 'Upload failed. Please try again.' },
      { status: 500 }
    );
  }
}

4. Create upload component in /components/papers/PaperUpload.tsx:

'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { toast } from 'react-hot-toast';
import { Upload, FileText, Loader2 } from 'lucide-react';

export function PaperUpload() {
  const [file, setFile] = useState<File | null>(null);
  const [uploading, setUploading] = useState(false);
  const router = useRouter();

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const selectedFile = e.target.files?.[0];
    if (selectedFile) {
      setFile(selectedFile);
    }
  };

  const handleUpload = async () => {
    if (!file) return;

    setUploading(true);
    const formData = new FormData();
    formData.append('file', file);

    try {
      const response = await fetch('/api/papers/upload', {
        method: 'POST',
        body: formData
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error || 'Upload failed');
      }

      toast.success('Paper uploaded successfully!');
      router.push(`/papers/${data.paper.id}`);
    } catch (error) {
      toast.error(error instanceof Error ? error.message : 'Upload failed');
    } finally {
      setUploading(false);
    }
  };

  return (
    <div className="max-w-2xl mx-auto p-6 border-2 border-dashed rounded-lg">
      <div className="text-center">
        <Upload className="mx-auto h-12 w-12 text-gray-400" />
        <h3 className="mt-2 text-lg font-medium">Upload Your Paper</h3>
        <p className="mt-1 text-sm text-gray-500">
          PDF or DOCX, up to 50MB
        </p>
      </div>

      <div className="mt-6">
        <input
          type="file"
          accept=".pdf,.docx"
          onChange={handleFileChange}
          className="hidden"
          id="file-upload"
        />
        <label htmlFor="file-upload">
          <Button variant="outline" className="w-full" asChild>
            <span>
              <FileText className="mr-2 h-4 w-4" />
              Choose File
            </span>
          </Button>
        </label>

        {file && (
          <div className="mt-4 p-4 bg-gray-50 rounded">
            <p className="text-sm font-medium">{file.name}</p>
            <p className="text-xs text-gray-500">
              {(file.size / 1024 / 1024).toFixed(2)} MB
            </p>
          </div>
        )}

        <Button
          onClick={handleUpload}
          disabled={!file || uploading}
          className="w-full mt-4"
        >
          {uploading ? (
            <>
              <Loader2 className="mr-2 h-4 w-4 animate-spin" />
              Uploading...
            </>
          ) : (
            'Upload Paper'
          )}
        </Button>
      </div>
    </div>
  );
}

5. Test upload flow:
   - Upload small PDF (< 5 pages)
   - Upload large PDF (50+ pages)
   - Upload DOCX file
   - Test file size limit (try 51MB file)
   - Test invalid file type
   - Verify text extraction quality

DELIVERABLES:
- PDF/DOCX text extraction working
- Upload API route functional
- Upload UI component complete
- Language detection working
- File validation in place
- All edge cases handled

TESTING CHECKLIST:
- [ ] Can upload PDF successfully
- [ ] Can upload DOCX successfully
- [ ] Rejected if > 50MB
- [ ] Rejected if wrong file type
- [ ] Text extracted correctly (check Uzbek/Cyrillic)
- [ ] Page count accurate
- [ ] Language detected correctly
```

---

## **DAY 4: FEEDBACK GENERATION - CONTENT ANALYSIS**

```
TASK: Build AI-powered content analysis using GPT-4o

1. Create OpenAI helper in /lib/openai.ts:

import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

export interface ContentFeedback {
  overall_assessment: {
    score: number;
    strengths: string[];
    weaknesses: string[];
  };
  section_feedback: {
    [key: string]: {
      score: number;
      comments: string;
      suggestions: string[];
    };
  };
  academic_quality: {
    research_question_clarity: number;
    argument_coherence: number;
    evidence_quality: number;
    critical_analysis: number;
  };
  language_quality: {
    grammar: number;
    academic_tone: number;
    terminology_accuracy: number;
  };
  specific_issues: Array<{
    location: string;
    issue: string;
    severity: 'high' | 'medium' | 'low';
    suggestion: string;
  }>;
  priority_improvements: string[];
}

export async function generateContentFeedback(
  paperText: string,
  title: string,
  pageCount: number,
  language: string
): Promise<{ feedback: ContentFeedback; tokensUsed: number; cost: number }> {
  
  const systemPrompt = `You are an expert Uzbek academic paper reviewer specializing in university-level research papers. You provide detailed, constructive feedback to help students improve their academic writing.`;

  const userPrompt = `
Analyze this ${language} research paper and provide detailed feedback on content quality.

**Paper Metadata:**
- Title: ${title}
- Length: ${pageCount} pages
- Language: ${language}

**Paper Content:**
${paperText.slice(0, 100000)} // Limit to ~25k tokens

**Provide feedback in JSON format following this structure:**
{
  "overall_assessment": {
    "score": 0-100,
    "strengths": ["strength 1", "strength 2", "strength 3"],
    "weaknesses": ["weakness 1", "weakness 2", "weakness 3"]
  },
  "section_feedback": {
    "introduction": {
      "score": 0-100,
      "comments": "Detailed feedback about the introduction",
      "suggestions": ["suggestion 1", "suggestion 2"]
    },
    "literature_review": {
      "score": 0-100,
      "comments": "Detailed feedback",
      "suggestions": ["suggestion 1", "suggestion 2"]
    },
    "methodology": {
      "score": 0-100,
      "comments": "Detailed feedback",
      "suggestions": ["suggestion 1", "suggestion 2"]
    },
    "results": {
      "score": 0-100,
      "comments": "Detailed feedback",
      "suggestions": ["suggestion 1", "suggestion 2"]
    },
    "conclusion": {
      "score": 0-100,
      "comments": "Detailed feedback",
      "suggestions": ["suggestion 1", "suggestion 2"]
    }
  },
  "academic_quality": {
    "research_question_clarity": 0-100,
    "argument_coherence": 0-100,
    "evidence_quality": 0-100,
    "critical_analysis": 0-100
  },
  "language_quality": {
    "grammar": 0-100,
    "academic_tone": 0-100,
    "terminology_accuracy": 0-100
  },
  "specific_issues": [
    {
      "location": "Introduction, paragraph 2",
      "issue": "Research question is unclear",
      "severity": "high",
      "suggestion": "Rephrase to explicitly state what you are investigating"
    }
  ],
  "priority_improvements": [
    "Most important thing to fix",
    "Second priority",
    "Third priority"
  ]
}

Output feedback in ${language === 'uzbek' ? 'Uzbek' : language === 'russian' ? 'Russian' : 'English'} language.
Be specific, constructive, and actionable in your feedback.
  `;

  const completion = await openai.chat.completions.create({
    model: 'gpt-4o-2024-08-06',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userPrompt }
    ],
    response_format: { type: 'json_object' },
    temperature: 0.3,
  });

  const feedback = JSON.parse(completion.choices[0].message.content!) as ContentFeedback;
  const tokensUsed = completion.usage!.total_tokens;
  const cost = calculateCost('gpt-4o', completion.usage!.prompt_tokens, completion.usage!.completion_tokens);

  return { feedback, tokensUsed, cost };
}

function calculateCost(model: string, inputTokens: number, outputTokens: number): number {
  const pricing = {
    'gpt-4o': { input: 2.50, output: 10.00 },
    'gpt-4o-mini': { input: 0.150, output: 0.600 }
  };
  
  const prices = pricing[model];
  return (inputTokens / 1_000_000 * prices.input) + (outputTokens / 1_000_000 * prices.output);
}

2. Create feedback API route in /app/api/papers/[id]/feedback/route.ts:

import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { generateContentFeedback } from '@/lib/openai';

export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const supabase = createRouteHandlerClient({ cookies });
    const { data: { user } } = await supabase.auth.getUser();
    
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const paperId = params.id;
    const { roundNumber } = await request.json();

    // 1. Get paper
    const { data: paper, error: paperError } = await supabase
      .from('papers')
      .select('*')
      .eq('id', paperId)
      .eq('user_id', user.id)
      .single();

    if (paperError || !paper) {
      return NextResponse.json({ error: 'Paper not found' }, { status: 404 });
    }

    // 2. Check if round already exists
    const { data: existingRound } = await supabase
      .from('feedback_rounds')
      .select('id')
      .eq('paper_id', paperId)
      .eq('round_number', roundNumber)
      .single();

    if (existingRound) {
      return NextResponse.json(
        { error: 'Feedback for this round already exists' },
        { status: 400 }
      );
    }

    // 3. Update paper status
    await supabase
      .from('papers')
      .update({ status: 'processing' })
      .eq('id', paperId);

    // 4. Generate content feedback
    const { feedback, tokensUsed, cost } = await generateContentFeedback(
      paper.extracted_text,
      paper.title,
      paper.page_count,
      paper.language
    );

    // 5. Save feedback to database
    const { data: feedbackRound, error: feedbackError } = await supabase
      .from('feedback_rounds')
      .insert({
        paper_id: paperId,
        round_number: roundNumber,
        model_used: 'gpt-4o',
        content_feedback: feedback,
        tokens_used: tokensUsed,
        cost_usd: cost
      })
      .select()
      .single();

    if (feedbackError) {
      throw new Error(`Failed to save feedback: ${feedbackError.message}`);
    }

    // 6. Update paper status to completed
    await supabase
      .from('papers')
      .update({ status: 'completed' })
      .eq('id', paperId);

    // 7. Decrement user's paper count (only for round 1)
    if (roundNumber === 1) {
      await supabase.rpc('decrement_papers_remaining', { 
        user_uuid: user.id 
      });
    }

    // 8. Log usage
    await supabase.from('usage_logs').insert({
      user_id: user.id,
      paper_id: paperId,
      action: `feedback_round_${roundNumber}`,
      tokens_used: tokensUsed,
      cost_usd: cost
    });

    return NextResponse.json({
      success: true,
      feedback: feedbackRound.content_feedback,
      roundNumber,
      tokensUsed,
      cost
    });

  } catch (error) {
    console.error('Feedback generation error:', error);
    
    // Update paper status to failed
    const supabase = createRouteHandlerClient({ cookies });
    await supabase
      .from('papers')
      .update({ status: 'failed' })
      .eq('id', params.id);

    return NextResponse.json(
      { error: 'Failed to generate feedback' },
      { status: 500 }
    );
  }
}

3. Create database function for decrementing papers:

-- In Supabase SQL Editor
CREATE OR REPLACE FUNCTION decrement_papers_remaining(user_uuid UUID)
RETURNS void AS $$
BEGIN
  UPDATE subscriptions
  SET papers_remaining = GREATEST(papers_remaining - 1, 0)
  WHERE user_id = user_uuid;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

4. Create feedback display component in /components/papers/FeedbackView.tsx:

'use client';

import { ContentFeedback } from '@/lib/openai';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Badge } from '@/components/ui/badge';
import { CheckCircle, AlertCircle, XCircle } from 'lucide-react';

interface FeedbackViewProps {
  feedback: ContentFeedback;
  roundNumber: number;
}

export function FeedbackView({ feedback, roundNumber }: FeedbackViewProps) {
  const getScoreColor = (score: number) => {
    if (score >= 80) return 'text-green-600';
    if (score >= 60) return 'text-yellow-600';
    return 'text-red-600';
  };

  const getSeverityIcon = (severity: string) => {
    switch (severity) {
      case 'high': return <XCircle className="h-4 w-4 text-red-500" />;
      case 'medium': return <AlertCircle className="h-4 w-4 text-yellow-500" />;
      case 'low': return <CheckCircle className="h-4 w-4 text-blue-500" />;
    }
  };

  return (
    <div className="space-y-6">
      {/* Overall Assessment */}
      <Card>
        <CardHeader>
          <CardTitle>Overall Assessment - Round {roundNumber}</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="flex items-center justify-between mb-4">
            <span className="text-sm font-medium">Overall Score</span>
            <span className={`text-3xl font-bold ${getScoreColor(feedback.overall_assessment.score)}`}>
              {feedback.overall_assessment.score}/100
            </span>
          </div>
          <Progress value={feedback.overall_assessment.score} className="mb-6" />

          <div className="grid md:grid-cols-2 gap-4">
            <div>
              <h4 className="font-medium text-green-600 mb-2">✅ Strengths</h4>
              <ul className="space-y-1">
                {feedback.overall_assessment.strengths.map((strength, i) => (
                  <li key={i} className="text-sm">{strength}</li>
                ))}
              </ul>
            </div>
            <div>
              <h4 className="font-medium text-red-600 mb-2">⚠️ Weaknesses</h4>
              <ul className="space-y-1">
                {feedback.overall_assessment.weaknesses.map((weakness, i) => (
                  <li key={i} className="text-sm">{weakness}</li>
                ))}
              </ul>
            </div>
          </div>
        </CardContent>
      </Card>

      {/* Section-by-Section Feedback */}
      <Card>
        <CardHeader>
          <CardTitle>Section Analysis</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          {Object.entries(feedback.section_feedback).map(([section, data]) => (
            <div key={section} className="border-b pb-4 last:border-0">
              <div className="flex items-center justify-between mb-2">
                <h4 className="font-medium capitalize">{section.replace('_', ' ')}</h4>
                <Badge variant={data.score >= 70 ? 'default' : 'destructive'}>
                  {data.score}/100
                </Badge>
              </div>
              <p className="text-sm text-gray-600 mb-2">{data.comments}</p>
              {data.suggestions.length > 0 && (
                <div>
                  <p className="text-xs font-medium text-gray-500 mb-1">Suggestions:</p>
                  <ul className="text-xs space-y-1">
                    {data.suggestions.map((suggestion, i) => (
                      <li key={i}>• {suggestion}</li>
                    ))}
                  </ul>
                </div>
              )}
            </div>
          ))}
        </CardContent>
      </Card>

      {/* Specific Issues */}
      <Card>
        <CardHeader>
          <CardTitle>Specific Issues Found</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-3">
            {feedback.specific_issues.map((issue, i) => (
              <div key={i} className="flex gap-3 p-3 bg-gray-50 rounded">
                {getSeverityIcon(issue.severity)}
                <div className="flex-1">
                  <div className="flex items-center gap-2 mb-1">
                    <span className="text-xs font-medium text-gray-500">{issue.location}</span>
                    <Badge variant="outline" className="text-xs">
                      {issue.severity}
                    </Badge>
                  </div>
                  <p className="text-sm font-medium">{issue.issue}</p>
                  <p className="text-xs text-gray-600 mt-1">💡 {issue.suggestion}</p>
                </div>
              </div>
            ))}
          </div>
        </CardContent>
      </Card>

      {/* Priority Improvements */}
      <Card>
        <CardHeader>
          <CardTitle>🎯 Top Priority Improvements</CardTitle>
        </CardHeader>
        <CardContent>
          <ol className="space-y-2">
            {feedback.priority_improvements.map((improvement, i) => (
              <li key={i} className="flex gap-2">
                <span className="font-bold text-blue-600">{i + 1}.</span>
                <span className="text-sm">{improvement}</span>
              </li>
            ))}
          </ol>
        </CardContent>
      </Card>
    </div>
  );
}

5. Test feedback generation:
   - Upload test paper
   - Generate Round 1 feedback
   - Verify JSON structure correct
   - Check scores make sense
   - Verify feedback is in correct language
   - Check token usage tracking

DELIVERABLES:
- Content feedback generation working
- API route functional
- Feedback display UI complete
- Token/cost tracking accurate
- Error handling in place

TESTING:
- [ ] Feedback generated successfully
- [ ] Scores between 0-100
- [ ] Feedback in correct language
- [ ] Token count accurate
- [ ] Cost calculation correct
- [ ] Database records saved
```

---

# 💻 DAYS 5-10 CURSOR PROMPTS (CONTINUED)

---

## **DAY 5: STRUCTURE VALIDATION & FEEDBACK**

```
TASK: Build Uzbek academic paper structure validation and compliance checking

1. Create structure template document in /lib/uzbek-structure.ts:

export const UZBEK_PAPER_STRUCTURE = {
  required_sections: [
    {
      name: 'title_page',
      uzbek: 'Muqova sahifa',
      required_elements: [
        'uzbek_title',
        'english_title',
        'author_info',
        'university_name',
        'year'
      ]
    },
    {
      name: 'uzbek_abstract',
      uzbek: 'Annotatsiya',
      min_words: 150,
      max_words: 250,
      required_elements: ['keywords'],
      keyword_count: { min: 5, max: 7 }
    },
    {
      name: 'russian_abstract',
      uzbek: 'Аннотация',
      min_words: 150,
      max_words: 250,
      required_elements: ['keywords']
    },
    {
      name: 'introduction',
      uzbek: 'Kirish',
      keywords: ['kirish', 'introduction', 'dolzarbligi', 'maqsad', 'vazifa'],
      required_elements: [
        'research_problem',
        'relevance',
        'objectives'
      ]
    },
    {
      name: 'literature_review',
      uzbek: 'Adabiyotlar tahlili',
      keywords: ['adabiyotlar', 'literature', 'tahlili', 'review', 'mavjud'],
      min_references: 10
    },
    {
      name: 'methodology',
      uzbek: 'Tadqiqot metodologiyasi',
      keywords: ['metodologiya', 'methodology', 'usul', 'method']
    },
    {
      name: 'results',
      uzbek: 'Natijalar va muhokama',
      keywords: ['natija', 'result', 'muhokama', 'discussion']
    },
    {
      name: 'conclusion',
      uzbek: 'Xulosa',
      keywords: ['xulosa', 'conclusion', 'yakun']
    },
    {
      name: 'references',
      uzbek: 'Foydalanilgan adabiyotlar',
      keywords: ['adabiyotlar', 'references', 'foydalanilgan'],
      min_count: 15
    }
  ],
  
  citation_format: {
    book: '{author}. {title}. – {publisher}: {city}, {year}. – {pages} b.',
    article: '{author}. {title} // {journal}. – {year}. – № {issue}. – B. {pages}.',
    online: '{author}. {title}. [Elektron resurs] – URL: {url} ({date}).'
  }
};

export interface StructureDetection {
  detected_sections: Array<{
    name: string;
    found: boolean;
    location?: string;
    word_count?: number;
  }>;
  missing_sections: string[];
  order_correct: boolean;
  compliance_score: number;
}

export function detectStructure(paperText: string): StructureDetection {
  const sections = UZBEK_PAPER_STRUCTURE.required_sections;
  const detected: StructureDetection['detected_sections'] = [];
  
  // Split paper into rough sections
  const textLower = paperText.toLowerCase();
  
  sections.forEach(section => {
    let found = false;
    let location = '';
    
    // Search for section keywords
    for (const keyword of section.keywords || []) {
      const regex = new RegExp(`\\b${keyword}\\b`, 'i');
      if (regex.test(textLower)) {
        found = true;
        const match = textLower.indexOf(keyword.toLowerCase());
        location = `Character ${match}`;
        break;
      }
    }
    
    detected.push({
      name: section.name,
      found,
      location: found ? location : undefined
    });
  });
  
  const missing_sections = detected
    .filter(s => !s.found)
    .map(s => s.name);
  
  const compliance_score = Math.round(
    (detected.filter(s => s.found).length / sections.length) * 100
  );
  
  return {
    detected_sections: detected,
    missing_sections,
    order_correct: checkSectionOrder(detected),
    compliance_score
  };
}

function checkSectionOrder(detected: StructureDetection['detected_sections']): boolean {
  // Verify sections appear in correct order
  const expectedOrder = [
    'title_page',
    'uzbek_abstract',
    'introduction',
    'literature_review',
    'methodology',
    'results',
    'conclusion',
    'references'
  ];
  
  const foundSections = detected
    .filter(s => s.found)
    .map(s => s.name);
  
  // Check if found sections maintain expected order
  let lastIndex = -1;
  for (const section of foundSections) {
    const currentIndex = expectedOrder.indexOf(section);
    if (currentIndex <= lastIndex) return false;
    lastIndex = currentIndex;
  }
  
  return true;
}

2. Create structure feedback prompt in /lib/openai.ts (add new function):

export interface StructureFeedback {
  compliance_score: number;
  structure_analysis: {
    correct_sections: string[];
    missing_sections: string[];
    incorrect_order: string[];
    formatting_issues: Array<{
      section: string;
      issue: string;
      fix: string;
    }>;
  };
  title_page: {
    has_uzbek_title: boolean;
    has_english_title: boolean;
    has_author_info: boolean;
    issues: string[];
  };
  abstract_analysis: {
    uzbek_abstract_word_count: number;
    has_keywords: boolean;
    keyword_count: number;
    issues: string[];
  };
  reference_format: {
    total_references: number;
    format_compliance: number;
    common_errors: string[];
  };
  priority_fixes: string[];
}

export async function generateStructureFeedback(
  paperText: string,
  detectedStructure: StructureDetection,
  language: string
): Promise<{ feedback: StructureFeedback; tokensUsed: number; cost: number }> {
  
  const systemPrompt = `You are an expert in Uzbek university research paper formatting standards. You analyze paper structure and provide detailed compliance feedback.`;

  const userPrompt = `
**Official Uzbek Paper Structure Required:**
${JSON.stringify(UZBEK_PAPER_STRUCTURE, null, 2)}

**Detected Structure in Submitted Paper:**
${JSON.stringify(detectedStructure, null, 2)}

**Paper Text (first 50,000 chars):**
${paperText.slice(0, 50000)}

**Task:** Analyze structure compliance and provide detailed formatting feedback.

**Output JSON following this exact structure:**
{
  "compliance_score": 0-100,
  "structure_analysis": {
    "correct_sections": ["array of sections present"],
    "missing_sections": ["array of missing sections"],
    "incorrect_order": ["section X should come before section Y"],
    "formatting_issues": [
      {
        "section": "Annotatsiya",
        "issue": "Word count is 89, should be 150-250",
        "fix": "Expand abstract with methodology details and key findings"
      }
    ]
  },
  "title_page": {
    "has_uzbek_title": true/false,
    "has_english_title": true/false,
    "has_author_info": true/false,
    "issues": ["array of specific issues"]
  },
  "abstract_analysis": {
    "uzbek_abstract_word_count": number,
    "has_keywords": true/false,
    "keyword_count": number,
    "issues": ["array of issues"]
  },
  "reference_format": {
    "total_references": number,
    "format_compliance": 0-100,
    "common_errors": ["Error 1", "Error 2"]
  },
  "priority_fixes": [
    "Most critical structural issue to fix",
    "Second priority",
    "Third priority"
  ]
}

**Important:**
- Be very specific about what's missing or wrong
- Provide actionable fixes for each issue
- Check citation format compliance carefully
- Output feedback in ${language === 'uzbek' ? 'Uzbek' : language === 'russian' ? 'Russian' : 'English'} language
  `;

  const completion = await openai.chat.completions.create({
    model: 'gpt-4o-2024-08-06',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userPrompt }
    ],
    response_format: { type: 'json_object' },
    temperature: 0.2,
  });

  const feedback = JSON.parse(completion.choices[0].message.content!) as StructureFeedback;
  const tokensUsed = completion.usage!.total_tokens;
  const cost = calculateCost('gpt-4o', completion.usage!.prompt_tokens, completion.usage!.completion_tokens);

  return { feedback, tokensUsed, cost };
}

3. Update feedback API route to include structure feedback in /app/api/papers/[id]/feedback/route.ts:

// Add this after generating content feedback (around line 50)

// 4b. Detect structure
import { detectStructure } from '@/lib/uzbek-structure';
const structureDetection = detectStructure(paper.extracted_text);

// 4c. Generate structure feedback
import { generateStructureFeedback } from '@/lib/openai';
const { 
  feedback: structureFeedback, 
  tokensUsed: structureTokens, 
  cost: structureCost 
} = await generateStructureFeedback(
  paper.extracted_text,
  structureDetection,
  paper.language
);

// 5. Save both feedbacks to database (update insert)
const { data: feedbackRound, error: feedbackError } = await supabase
  .from('feedback_rounds')
  .insert({
    paper_id: paperId,
    round_number: roundNumber,
    model_used: 'gpt-4o',
    content_feedback: feedback,
    structure_feedback: structureFeedback, // ADD THIS
    tokens_used: tokensUsed + structureTokens, // UPDATE THIS
    cost_usd: cost + structureCost // UPDATE THIS
  })
  .select()
  .single();

4. Create structure feedback component in /components/papers/StructureFeedbackView.tsx:

'use client';

import { StructureFeedback } from '@/lib/openai';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { CheckCircle2, XCircle, AlertTriangle } from 'lucide-react';

interface StructureFeedbackViewProps {
  feedback: StructureFeedback;
}

export function StructureFeedbackView({ feedback }: StructureFeedbackViewProps) {
  return (
    <div className="space-y-6">
      {/* Compliance Score */}
      <Card>
        <CardHeader>
          <CardTitle>Structure Compliance</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="flex items-center justify-between">
            <span className="text-sm">Compliance Score</span>
            <div className="flex items-center gap-2">
              <span className={`text-3xl font-bold ${
                feedback.compliance_score >= 80 ? 'text-green-600' :
                feedback.compliance_score >= 60 ? 'text-yellow-600' :
                'text-red-600'
              }`}>
                {feedback.compliance_score}%
              </span>
              {feedback.compliance_score >= 80 ? (
                <CheckCircle2 className="h-8 w-8 text-green-600" />
              ) : (
                <AlertTriangle className="h-8 w-8 text-yellow-600" />
              )}
            </div>
          </div>
        </CardContent>
      </Card>

      {/* Section Analysis */}
      <Card>
        <CardHeader>
          <CardTitle>Section Analysis</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          {/* Correct Sections */}
          {feedback.structure_analysis.correct_sections.length > 0 && (
            <div>
              <h4 className="text-sm font-medium text-green-600 flex items-center gap-2 mb-2">
                <CheckCircle2 className="h-4 w-4" />
                Present Sections ({feedback.structure_analysis.correct_sections.length})
              </h4>
              <div className="flex flex-wrap gap-2">
                {feedback.structure_analysis.correct_sections.map(section => (
                  <Badge key={section} variant="outline" className="bg-green-50">
                    {section}
                  </Badge>
                ))}
              </div>
            </div>
          )}

          {/* Missing Sections */}
          {feedback.structure_analysis.missing_sections.length > 0 && (
            <div>
              <h4 className="text-sm font-medium text-red-600 flex items-center gap-2 mb-2">
                <XCircle className="h-4 w-4" />
                Missing Sections ({feedback.structure_analysis.missing_sections.length})
              </h4>
              <div className="flex flex-wrap gap-2">
                {feedback.structure_analysis.missing_sections.map(section => (
                  <Badge key={section} variant="destructive">
                    {section}
                  </Badge>
                ))}
              </div>
            </div>
          )}

          {/* Order Issues */}
          {feedback.structure_analysis.incorrect_order.length > 0 && (
            <div>
              <h4 className="text-sm font-medium text-yellow-600 flex items-center gap-2 mb-2">
                <AlertTriangle className="h-4 w-4" />
                Section Order Issues
              </h4>
              <ul className="text-sm space-y-1">
                {feedback.structure_analysis.incorrect_order.map((issue, i) => (
                  <li key={i} className="text-yellow-700">• {issue}</li>
                ))}
              </ul>
            </div>
          )}

          {/* Formatting Issues */}
          {feedback.structure_analysis.formatting_issues.length > 0 && (
            <div>
              <h4 className="text-sm font-medium mb-3">Formatting Issues</h4>
              <div className="space-y-3">
                {feedback.structure_analysis.formatting_issues.map((item, i) => (
                  <div key={i} className="border-l-4 border-yellow-400 pl-4 py-2 bg-yellow-50">
                    <div className="font-medium text-sm">{item.section}</div>
                    <div className="text-sm text-gray-700 mt-1">
                      <span className="text-red-600">Issue:</span> {item.issue}
                    </div>
                    <div className="text-sm text-gray-700 mt-1">
                      <span className="text-green-600">Fix:</span> {item.fix}
                    </div>
                  </div>
                ))}
              </div>
            </div>
          )}
        </CardContent>
      </Card>

      {/* Title Page Check */}
      <Card>
        <CardHeader>
          <CardTitle>Title Page</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-2">
            <div className="flex items-center gap-2">
              {feedback.title_page.has_uzbek_title ? (
                <CheckCircle2 className="h-4 w-4 text-green-600" />
              ) : (
                <XCircle className="h-4 w-4 text-red-600" />
              )}
              <span className="text-sm">Uzbek Title</span>
            </div>
            <div className="flex items-center gap-2">
              {feedback.title_page.has_english_title ? (
                <CheckCircle2 className="h-4 w-4 text-green-600" />
              ) : (
                <XCircle className="h-4 w-4 text-red-600" />
              )}
              <span className="text-sm">English Title</span>
            </div>
            <div className="flex items-center gap-2">
              {feedback.title_page.has_author_info ? (
                <CheckCircle2 className="h-4 w-4 text-green-600" />
              ) : (
                <XCircle className="h-4 w-4 text-red-600" />
              )}
              <span className="text-sm">Author Information</span>
            </div>
            {feedback.title_page.issues.length > 0 && (
              <div className="mt-3 p-3 bg-red-50 rounded">
                <ul className="text-sm space-y-1">
                  {feedback.title_page.issues.map((issue, i) => (
                    <li key={i} className="text-red-700">• {issue}</li>
                  ))}
                </ul>
              </div>
            )}
          </div>
        </CardContent>
      </Card>

      {/* Abstract Analysis */}
      <Card>
        <CardHeader>
          <CardTitle>Abstract (Annotatsiya)</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-3">
            <div className="flex justify-between items-center">
              <span className="text-sm">Word Count</span>
              <Badge variant={
                feedback.abstract_analysis.uzbek_abstract_word_count >= 150 &&
                feedback.abstract_analysis.uzbek_abstract_word_count <= 250
                  ? 'default' : 'destructive'
              }>
                {feedback.abstract_analysis.uzbek_abstract_word_count} words
                {feedback.abstract_analysis.uzbek_abstract_word_count < 150 && ' (Too short)'}
                {feedback.abstract_analysis.uzbek_abstract_word_count > 250 && ' (Too long)'}
              </Badge>
            </div>
            
            <div className="flex justify-between items-center">
              <span className="text-sm">Keywords</span>
              <Badge variant={
                feedback.abstract_analysis.keyword_count >= 5 &&
                feedback.abstract_analysis.keyword_count <= 7
                  ? 'default' : 'destructive'
              }>
                {feedback.abstract_analysis.keyword_count} keywords
                {feedback.abstract_analysis.keyword_count < 5 && ' (Need 5-7)'}
                {feedback.abstract_analysis.keyword_count > 7 && ' (Too many)'}
              </Badge>
            </div>

            {feedback.abstract_analysis.issues.length > 0 && (
              <div className="p-3 bg-yellow-50 rounded">
                <ul className="text-sm space-y-1">
                  {feedback.abstract_analysis.issues.map((issue, i) => (
                    <li key={i}>• {issue}</li>
                  ))}
                </ul>
              </div>
            )}
          </div>
        </CardContent>
      </Card>

      {/* References */}
      <Card>
        <CardHeader>
          <CardTitle>References (Foydalanilgan adabiyotlar)</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-3">
            <div className="flex justify-between items-center">
              <span className="text-sm">Total References</span>
              <Badge variant={feedback.reference_format.total_references >= 15 ? 'default' : 'destructive'}>
                {feedback.reference_format.total_references}
                {feedback.reference_format.total_references < 15 && ' (Minimum 15 required)'}
              </Badge>
            </div>

            <div className="flex justify-between items-center">
              <span className="text-sm">Format Compliance</span>
              <span className={`font-bold ${
                feedback.reference_format.format_compliance >= 80 ? 'text-green-600' :
                feedback.reference_format.format_compliance >= 60 ? 'text-yellow-600' :
                'text-red-600'
              }`}>
                {feedback.reference_format.format_compliance}%
              </span>
            </div>

            {feedback.reference_format.common_errors.length > 0 && (
              <div>
                <h5 className="text-sm font-medium mb-2">Common Errors:</h5>
                <ul className="text-sm space-y-1 text-gray-700">
                  {feedback.reference_format.common_errors.map((error, i) => (
                    <li key={i}>• {error}</li>
                  ))}
                </ul>
              </div>
            )}
          </div>
        </CardContent>
      </Card>

      {/* Priority Fixes */}
      <Card>
        <CardHeader>
          <CardTitle>🎯 Priority Structure Fixes</CardTitle>
        </CardHeader>
        <CardContent>
          <ol className="space-y-2">
            {feedback.priority_fixes.map((fix, i) => (
              <li key={i} className="flex gap-2">
                <span className="font-bold text-blue-600">{i + 1}.</span>
                <span className="text-sm">{fix}</span>
              </li>
            ))}
          </ol>
        </CardContent>
      </Card>
    </div>
  );
}

5. Update paper detail page to show both content and structure feedback:

// /app/papers/[id]/page.tsx - add tabs for Content vs Structure feedback

import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { FeedbackView } from '@/components/papers/FeedbackView';
import { StructureFeedbackView } from '@/components/papers/StructureFeedbackView';

// In your render:
<Tabs defaultValue="content">
  <TabsList>
    <TabsTrigger value="content">Content Feedback</TabsTrigger>
    <TabsTrigger value="structure">Structure Feedback</TabsTrigger>
  </TabsList>
  <TabsContent value="content">
    <FeedbackView feedback={contentFeedback} roundNumber={1} />
  </TabsContent>
  <TabsContent value="structure">
    <StructureFeedbackView feedback={structureFeedback} />
  </TabsContent>
</Tabs>

DELIVERABLES:
- Structure detection algorithm working
- Structure feedback generation complete
- Combined feedback display
- Citation format validation
- All checks functional

TESTING:
- [ ] Detects all sections correctly
- [ ] Identifies missing sections
- [ ] Checks abstract word count
- [ ] Validates keywords (5-7)
- [ ] Checks reference count (15+)
- [ ] Identifies citation format errors
```

---

## **DAY 6: BEFORE/AFTER COMPARISON VIEW**

```
TASK: Build visual diff comparison showing original vs. improved versions

1. Install diff library:
   npm install diff
   npm install react-diff-viewer-continued

2. Create comparison utility in /lib/diff-utils.ts:

import * as Diff from 'diff';

export interface TextChange {
  type: 'added' | 'removed' | 'unchanged';
  value: string;
  lineNumber?: number;
}

export function generateDiff(originalText: string, revisedText: string): TextChange[] {
  const changes = Diff.diffWords(originalText, revisedText);
  
  return changes.map(change => ({
    type: change.added ? 'added' : change.removed ? 'removed' : 'unchanged',
    value: change.value
  }));
}

export function countChanges(changes: TextChange[]) {
  return {
    additions: changes.filter(c => c.type === 'added').length,
    deletions: changes.filter(c => c.type === 'removed').length,
    total: changes.filter(c => c.type !== 'unchanged').length
  };
}

export interface SectionComparison {
  section_name: string;
  original: string;
  revised: string;
  changes: TextChange[];
  improvement_score: number;
}

export function compareSections(
  originalText: string,
  revisedText: string,
  sectionNames: string[]
): SectionComparison[] {
  // This is simplified - in production, use proper section detection
  const comparisons: SectionComparison[] = [];
  
  for (const sectionName of sectionNames) {
    // Extract section from both texts (simplified)
    const originalSection = extractSection(originalText, sectionName);
    const revisedSection = extractSection(revisedText, sectionName);
    
    if (originalSection && revisedSection) {
      const changes = generateDiff(originalSection, revisedSection);
      
      comparisons.push({
        section_name: sectionName,
        original: originalSection,
        revised: revisedSection,
        changes,
        improvement_score: calculateImprovementScore(changes)
      });
    }
  }
  
  return comparisons;
}

function extractSection(text: string, sectionName: string): string | null {
  // Simplified - you'd want more sophisticated section detection
  const regex = new RegExp(`${sectionName}[\\s\\S]{0,5000}`, 'i');
  const match = text.match(regex);
  return match ? match[0] : null;
}

function calculateImprovementScore(changes: TextChange[]): number {
  const stats = countChanges(changes);
  if (stats.total === 0) return 0;
  
  // Simple heuristic: more additions = better
  return Math.min(100, (stats.additions / stats.total) * 100);
}

3. Create API endpoint for generating revised version in /app/api/papers/[id]/revise/route.ts:

import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const supabase = createRouteHandlerClient({ cookies });
    const { data: { user } } = await supabase.auth.getUser();
    
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const { roundNumber } = await request.json();

    // Get paper and feedback
    const { data: paper } = await supabase
      .from('papers')
      .select('*, feedback_rounds(*)')
      .eq('id', params.id)
      .single();

    if (!paper) {
      return NextResponse.json({ error: 'Paper not found' }, { status: 404 });
    }

    const feedback = paper.feedback_rounds.find(
      (r: any) => r.round_number === roundNumber
    );

    if (!feedback) {
      return NextResponse.json({ error: 'Feedback not found' }, { status: 404 });
    }

    // Generate revised version using GPT-4o-mini
    const systemPrompt = `You are an academic writing editor. Generate an improved version of the paper based on the feedback provided.`;

    const userPrompt = `
**Original Paper:**
${paper.extracted_text}

**Content Feedback:**
${JSON.stringify(feedback.content_feedback, null, 2)}

**Structure Feedback:**
${JSON.stringify(feedback.structure_feedback, null, 2)}

**Task:** 
1. Apply all suggested improvements
2. Fix all identified issues
3. Maintain the original meaning and research content
4. Improve clarity and academic tone
5. Fix structure compliance issues
6. Return the COMPLETE revised paper text

**Important:** 
- Keep all sections in the correct order
- Maintain Uzbek language if original is Uzbek
- Preserve all citations and references
- Output ONLY the revised paper text, no explanations
    `;

    const completion = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: userPrompt }
      ],
      temperature: 0.3,
      max_tokens: 16000,
    });

    const revisedText = completion.choices[0].message.content!;
    const tokensUsed = completion.usage!.total_tokens;
    const cost = (tokensUsed / 1_000_000) * 0.600; // GPT-4o-mini output pricing

    // Update feedback round with revised text
    await supabase
      .from('feedback_rounds')
      .update({
        revised_text: revisedText,
        tokens_used: feedback.tokens_used + tokensUsed,
        cost_usd: feedback.cost_usd + cost
      })
      .eq('id', feedback.id);

    // Log usage
    await supabase.from('usage_logs').insert({
      user_id: user.id,
      paper_id: params.id,
      action: `generate_revision_round_${roundNumber}`,
      tokens_used: tokensUsed,
      cost_usd: cost
    });

    return NextResponse.json({
      success: true,
      revised_text: revisedText,
      tokensUsed,
      cost
    });

  } catch (error) {
    console.error('Revision generation error:', error);
    return NextResponse.json(
      { error: 'Failed to generate revision' },
      { status: 500 }
    );
  }
}

4. Create comparison view component in /components/papers/ComparisonView.tsx:

'use client';

import { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Badge } from '@/components/ui/badge';
import { generateDiff, countChanges, TextChange } from '@/lib/diff-utils';
import { Download, FileText, GitCompare } from 'lucide-react';

interface ComparisonViewProps {
  originalText: string;
  revisedText: string;
  paperId: string;
}

export function ComparisonView({ originalText, revisedText, paperId }: ComparisonViewProps) {
  const [viewMode, setViewMode] = useState<'split' | 'unified'>('split');
  const changes = generateDiff(originalText, revisedText);
  const stats = countChanges(changes);

  const handleExport = async (format: 'pdf' | 'docx') => {
    // Call export API
    const response = await fetch(`/api/papers/${paperId}/export`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ format, includeTracking: true })
    });
    
    const blob = await response.blob();
    const url = window.URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `revised_paper.${format}`;
    a.click();
  };

  return (
    <div className="space-y-6">
      {/* Statistics */}
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center gap-2">
            <GitCompare className="h-5 w-5" />
            Changes Summary
          </CardTitle>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-3 gap-4">
            <div className="text-center">
              <div className="text-3xl font-bold text-green-600">{stats.additions}</div>
              <div className="text-sm text-gray-500">Additions</div>
            </div>
            <div className="text-center">
              <div className="text-3xl font-bold text-red-600">{stats.deletions}</div>
              <div className="text-sm text-gray-500">Deletions</div>
            </div>
            <div className="text-center">
              <div className="text-3xl font-bold text-blue-600">{stats.total}</div>
              <div className="text-sm text-gray-500">Total Changes</div>
            </div>
          </div>

          <div className="mt-4 flex gap-2">
            <Button onClick={() => handleExport('docx')} variant="outline" size="sm">
              <FileText className="mr-2 h-4 w-4" />
              Export DOCX
            </Button>
            <Button onClick={() => handleExport('pdf')} variant="outline" size="sm">
              <Download className="mr-2 h-4 w-4" />
              Export PDF
            </Button>
          </div>
        </CardContent>
      </Card>

      {/* View Mode Toggle */}
      <div className="flex justify-end">
        <Tabs value={viewMode} onValueChange={(v) => setViewMode(v as any)}>
          <TabsList>
            <TabsTrigger value="split">Split View</TabsTrigger>
            <TabsTrigger value="unified">Unified View</TabsTrigger>
          </TabsList>
        </Tabs>
      </div>

      {/* Comparison Display */}
      {viewMode === 'split' ? (
        <div className="grid md:grid-cols-2 gap-4">
          {/* Original */}
          <Card>
            <CardHeader>
              <CardTitle className="text-lg">Original Version</CardTitle>
            </CardHeader>
            <CardContent>
              <div className="prose prose-sm max-w-none">
                <pre className="whitespace-pre-wrap text-sm font-mono bg-gray-50 p-4 rounded">
                  {originalText}
                </pre>
              </div>
            </CardContent>
          </Card>

          {/* Revised */}
          <Card>
            <CardHeader>
              <CardTitle className="text-lg">Revised Version</CardTitle>
            </CardHeader>
            <CardContent>
              <div className="prose prose-sm max-w-none">
                <pre className="whitespace-pre-wrap text-sm font-mono bg-green-50 p-4 rounded">
                  {revisedText}
                </pre>
              </div>
            </CardContent>
          </Card>
        </div>
      ) : (
        <Card>
          <CardHeader>
            <CardTitle>Unified Diff View</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="font-mono text-sm space-y-1">
              {changes.map((change, i) => (
                <div
                  key={i}
                  className={`p-2 ${
                    change.type === 'added' ? 'bg-green-100 text-green-900' :
                    change.type === 'removed' ? 'bg-red-100 text-red-900 line-through' :
                    'bg-gray-50'
                  }`}
                >
                  {change.type === 'added' && '+ '}
                  {change.type === 'removed' && '- '}
                  {change.value}
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      )}

      {/* Change Categories */}
      <Card>
        <CardHeader>
          <CardTitle>Types of Changes Made</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-2">
            <div className="flex items-center gap-2">
              <Badge variant="outline" className="bg-green-50">Grammar Fixes</Badge>
              <span className="text-sm text-gray-600">12 corrections</span>
            </div>
            <div className="flex items-center gap-2">
              <Badge variant="outline" className="bg-blue-50">Content Additions</Badge>
              <span className="text-sm text-gray-600">5 sections expanded</span>
            </div>
            <div className="flex items-center gap-2">
              <Badge variant="outline" className="bg-yellow-50">Structure Changes</Badge>
              <span className="text-sm text-gray-600">2 sections reordered</span>
            </div>
            <div className="flex items-center gap-2">
              <Badge variant="outline" className="bg-purple-50">Citation Fixes</Badge>
              <span className="text-sm text-gray-600">8 references reformatted</span>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}

5. Add export functionality in /app/api/papers/[id]/export/route.ts:

import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { Document, Packer, Paragraph, TextRun } from 'docx';
import PDFDocument from 'pdfkit';

export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const supabase = createRouteHandlerClient({ cookies });
    const { data: { user } } = await supabase.auth.getUser();
    
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const { format, includeTracking } = await request.json();

    // Get paper with latest feedback
    const { data: paper } = await supabase
      .from('papers')
      .select('*, feedback_rounds(*)')
      .eq('id', params.id)
      .single();

    if (!paper) {
      return NextResponse.json({ error: 'Paper not found' }, { status: 404 });
    }

    const latestFeedback = paper.feedback_rounds
      .sort((a: any, b: any) => b.round_number - a.round_number)[0];

    const textToExport = latestFeedback?.revised_text || paper.extracted_text;

    if (format === 'docx') {
      // Create DOCX with track changes if requested
      const doc = new Document({
        sections: [{
          properties: {},
          children: [
            new Paragraph({
              children: [
                new TextRun({
                  text: textToExport,
                  font: 'Times New Roman',
                  size: 24, // 12pt
                })
              ]
            })
          ]
        }]
      });

      const buffer = await Packer.toBuffer(doc);
      
      return new NextResponse(buffer, {
        headers: {
          'Content-Type': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
          'Content-Disposition': 'attachment; filename="revised_paper.docx"'
        }
      });
    }

    if (format === 'pdf') {
      const doc = new PDFDocument();
      const chunks: Buffer[] = [];

      doc.on('data', (chunk) => chunks.push(chunk));
      doc.on('end', () => {});

      doc.fontSize(12).font('Times-Roman').text(textToExport);
      doc.end();

      await new Promise(resolve => doc.on('end', resolve));
      const buffer = Buffer.concat(chunks);

      return new NextResponse(buffer, {
        headers: {
          'Content-Type': 'application/pdf',
          'Content-Disposition': 'attachment; filename="revised_paper.pdf"'
        }
      });
    }

    return NextResponse.json({ error: 'Invalid format' }, { status: 400 });

  } catch (error) {
    console.error('Export error:', error);
    return NextResponse.json(
      { error: 'Export failed' },
      { status: 500 }
    );
  }
}

DELIVERABLES:
- Diff generation working
- Split/unified view toggle
- Export to DOCX/PDF
- Change statistics display
- Visual highlighting of changes

TESTING:
- [ ] Diff correctly shows additions/deletions
- [ ] Split view displays both versions
- [ ] Unified view shows inline changes
- [ ] Export DOCX works
- [ ] Export PDF works
- [ ] Statistics accurate
```

---

# 💻 DAYS 7-10 CURSOR PROMPTS (FINAL)

---

## **DAY 7: ROUNDS 2-3 REFINEMENT & ITERATIVE IMPROVEMENT**

```
TASK: Build cost-efficient refinement system using GPT-4o-mini for rounds 2-3

1. Create refinement prompt in /lib/openai.ts (add new function):

export interface RefinementFeedback {
  progress_score: number;
  issues_resolved: Array<{
    issue: string;
    original_location: string;
    resolution_quality: 'excellent' | 'good' | 'partial' | 'unresolved';
  }>;
  issues_remaining: Array<{
    issue: string;
    location: string;
    priority: 'high' | 'medium' | 'low';
    suggestion: string;
  }>;
  new_suggestions: string[];
  ready_for_submission: boolean;
  final_notes: string;
  overall_improvement: string; // "improved", "unchanged", "degraded"
}

export async function generateRefinementFeedback(
  originalText: string,
  previousFeedback: any,
  currentRevision: string,
  roundNumber: number,
  language: string
): Promise<{ feedback: RefinementFeedback; tokensUsed: number; cost: number }> {
  
  const systemPrompt = `You are an academic writing reviewer providing focused feedback on paper revisions. Be concise and specific.`;

  const userPrompt = `
**Original Paper:**
${originalText.slice(0, 30000)}

**Round ${roundNumber - 1} Feedback Summary:**
Priority Issues Identified:
${JSON.stringify(previousFeedback.priority_improvements || [], null, 2)}

Content Issues:
${JSON.stringify(previousFeedback.specific_issues?.slice(0, 10) || [], null, 2)}

Structure Issues:
${JSON.stringify(previousFeedback.priority_fixes?.slice(0, 5) || [], null, 2)}

**User's Revised Version:**
${currentRevision.slice(0, 30000)}

**Task:** Provide focused Round ${roundNumber} feedback:
1. Check if previous issues were addressed
2. Identify any new issues introduced
3. Provide final polish suggestions
4. Determine if paper is ready for submission

**Output JSON:**
{
  "progress_score": 0-100,
  "issues_resolved": [
    {
      "issue": "Abstract was too short (89 words)",
      "original_location": "Annotatsiya section",
      "resolution_quality": "excellent" // excellent|good|partial|unresolved
    }
  ],
  "issues_remaining": [
    {
      "issue": "Description of the issue",
      "location": "Specific section or page",
      "priority": "high", // high|medium|low
      "suggestion": "How to fix it"
    }
  ],
  "new_suggestions": [
    "Additional improvement suggestion 1",
    "Additional improvement suggestion 2"
  ],
  "ready_for_submission": true/false,
  "final_notes": "Brief summary of revision quality",
  "overall_improvement": "improved" // improved|unchanged|degraded
}

**Important:**
- Be CONCISE - focus only on remaining issues
- Compare current version with original to measure improvement
- Only flag HIGH priority issues if paper quality degraded
- Output in ${language === 'uzbek' ? 'Uzbek' : language === 'russian' ? 'Russian' : 'English'}
  `;

  const completion = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userPrompt }
    ],
    response_format: { type: 'json_object' },
    temperature: 0.2,
  });

  const feedback = JSON.parse(completion.choices[0].message.content!) as RefinementFeedback;
  const tokensUsed = completion.usage!.total_tokens;
  const cost = calculateCost('gpt-4o-mini', completion.usage!.prompt_tokens, completion.usage!.completion_tokens);

  return { feedback, tokensUsed, cost };
}

2. Update feedback API route to handle rounds 2-3 differently in /app/api/papers/[id]/feedback/route.ts:

// After checking round number, add branching logic:

if (roundNumber === 1) {
  // ROUND 1: Full analysis (existing code)
  const { feedback: contentFeedback, tokensUsed, cost } = await generateContentFeedback(
    paper.extracted_text,
    paper.title,
    paper.page_count,
    paper.language
  );

  const structureDetection = detectStructure(paper.extracted_text);
  const { 
    feedback: structureFeedback, 
    tokensUsed: structureTokens, 
    cost: structureCost 
  } = await generateStructureFeedback(
    paper.extracted_text,
    structureDetection,
    paper.language
  );

  // Save both feedbacks
  const { data: feedbackRound } = await supabase
    .from('feedback_rounds')
    .insert({
      paper_id: paperId,
      round_number: 1,
      model_used: 'gpt-4o',
      content_feedback: contentFeedback,
      structure_feedback: structureFeedback,
      tokens_used: tokensUsed + structureTokens,
      cost_usd: cost + structureCost
    })
    .select()
    .single();

  return NextResponse.json({
    success: true,
    feedback: {
      content: contentFeedback,
      structure: structureFeedback
    },
    roundNumber: 1,
    tokensUsed: tokensUsed + structureTokens,
    cost: cost + structureCost
  });

} else {
  // ROUNDS 2-3: Refinement (cheaper, focused)
  
  // Get previous round feedback
  const { data: previousRound } = await supabase
    .from('feedback_rounds')
    .select('*')
    .eq('paper_id', paperId)
    .eq('round_number', roundNumber - 1)
    .single();

  if (!previousRound) {
    return NextResponse.json(
      { error: 'Previous round not found. Complete Round 1 first.' },
      { status: 400 }
    );
  }

  // Get current revision (user should upload revised version)
  const { currentRevision } = await request.json();
  
  if (!currentRevision) {
    return NextResponse.json(
      { error: 'Please provide your revised paper text' },
      { status: 400 }
    );
  }

  // Generate refinement feedback
  import { generateRefinementFeedback } from '@/lib/openai';
  const { feedback, tokensUsed, cost } = await generateRefinementFeedback(
    paper.extracted_text,
    {
      ...previousRound.content_feedback,
      ...previousRound.structure_feedback
    },
    currentRevision,
    roundNumber,
    paper.language
  );

  // Save refinement feedback
  const { data: feedbackRound } = await supabase
    .from('feedback_rounds')
    .insert({
      paper_id: paperId,
      round_number: roundNumber,
      model_used: 'gpt-4o-mini',
      content_feedback: feedback,
      revised_text: currentRevision,
      tokens_used: tokensUsed,
      cost_usd: cost
    })
    .select()
    .single();

  return NextResponse.json({
    success: true,
    feedback,
    roundNumber,
    tokensUsed,
    cost,
    readyForSubmission: feedback.ready_for_submission
  });
}

3. Create refinement feedback component in /components/papers/RefinementFeedbackView.tsx:

'use client';

import { RefinementFeedback } from '@/lib/openai';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Progress } from '@/components/ui/progress';
import { CheckCircle2, Circle, AlertCircle, XCircle, PartyPopper } from 'lucide-react';
import { Button } from '@/components/ui/button';

interface RefinementFeedbackViewProps {
  feedback: RefinementFeedback;
  roundNumber: number;
  onNextRound?: () => void;
  onFinalize?: () => void;
}

export function RefinementFeedbackView({ 
  feedback, 
  roundNumber,
  onNextRound,
  onFinalize 
}: RefinementFeedbackViewProps) {
  
  const getResolutionIcon = (quality: string) => {
    switch (quality) {
      case 'excellent': return <CheckCircle2 className="h-4 w-4 text-green-600" />;
      case 'good': return <CheckCircle2 className="h-4 w-4 text-blue-600" />;
      case 'partial': return <AlertCircle className="h-4 w-4 text-yellow-600" />;
      case 'unresolved': return <XCircle className="h-4 w-4 text-red-600" />;
      default: return <Circle className="h-4 w-4" />;
    }
  };

  const getPriorityColor = (priority: string) => {
    switch (priority) {
      case 'high': return 'destructive';
      case 'medium': return 'secondary';
      case 'low': return 'outline';
      default: return 'default';
    }
  };

  return (
    <div className="space-y-6">
      {/* Progress Score */}
      <Card>
        <CardHeader>
          <CardTitle>Round {roundNumber} Progress</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-4">
            <div className="flex items-center justify-between">
              <span className="text-sm font-medium">Progress Score</span>
              <span className={`text-3xl font-bold ${
                feedback.progress_score >= 90 ? 'text-green-600' :
                feedback.progress_score >= 75 ? 'text-blue-600' :
                feedback.progress_score >= 60 ? 'text-yellow-600' :
                'text-red-600'
              }`}>
                {feedback.progress_score}/100
              </span>
            </div>
            <Progress value={feedback.progress_score} />

            <div className="flex items-center gap-2 p-3 rounded-lg bg-gray-50">
              {feedback.overall_improvement === 'improved' && (
                <>
                  <CheckCircle2 className="h-5 w-5 text-green-600" />
                  <span className="text-sm font-medium text-green-700">Paper Improved</span>
                </>
              )}
              {feedback.overall_improvement === 'unchanged' && (
                <>
                  <Circle className="h-5 w-5 text-gray-600" />
                  <span className="text-sm font-medium text-gray-700">No Significant Change</span>
                </>
              )}
              {feedback.overall_improvement === 'degraded' && (
                <>
                  <XCircle className="h-5 w-5 text-red-600" />
                  <span className="text-sm font-medium text-red-700">Quality Decreased</span>
                </>
              )}
            </div>

            {feedback.ready_for_submission && (
              <div className="flex items-center gap-2 p-4 rounded-lg bg-green-50 border-2 border-green-200">
                <PartyPopper className="h-6 w-6 text-green-600" />
                <div className="flex-1">
                  <p className="font-medium text-green-900">Ready for Submission! 🎉</p>
                  <p className="text-sm text-green-700">Your paper meets quality standards.</p>
                </div>
              </div>
            )}
          </div>
        </CardContent>
      </Card>

      {/* Issues Resolved */}
      {feedback.issues_resolved.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle className="text-green-600">✅ Issues Resolved ({feedback.issues_resolved.length})</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="space-y-3">
              {feedback.issues_resolved.map((item, i) => (
                <div key={i} className="flex gap-3 p-3 bg-green-50 rounded-lg">
                  {getResolutionIcon(item.resolution_quality)}
                  <div className="flex-1">
                    <p className="text-sm font-medium">{item.issue}</p>
                    <p className="text-xs text-gray-600 mt-1">{item.original_location}</p>
                    <Badge 
                      variant="outline" 
                      className="mt-2 text-xs"
                    >
                      {item.resolution_quality}
                    </Badge>
                  </div>
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      )}

      {/* Issues Remaining */}
      {feedback.issues_remaining.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle className="text-yellow-600">⚠️ Issues Remaining ({feedback.issues_remaining.length})</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="space-y-3">
              {feedback.issues_remaining.map((item, i) => (
                <div key={i} className="border-l-4 border-yellow-400 pl-4 py-2">
                  <div className="flex items-center justify-between mb-1">
                    <span className="text-sm font-medium">{item.issue}</span>
                    <Badge variant={getPriorityColor(item.priority) as any}>
                      {item.priority}
                    </Badge>
                  </div>
                  <p className="text-xs text-gray-600 mb-1">{item.location}</p>
                  <p className="text-sm text-gray-700 mt-2">
                    <span className="font-medium">💡 Fix:</span> {item.suggestion}
                  </p>
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      )}

      {/* New Suggestions */}
      {feedback.new_suggestions.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle>💡 Additional Suggestions</CardTitle>
          </CardHeader>
          <CardContent>
            <ul className="space-y-2">
              {feedback.new_suggestions.map((suggestion, i) => (
                <li key={i} className="flex gap-2 text-sm">
                  <span className="text-blue-600">•</span>
                  <span>{suggestion}</span>
                </li>
              ))}
            </ul>
          </CardContent>
        </Card>
      )}

      {/* Final Notes */}
      <Card>
        <CardHeader>
          <CardTitle>Reviewer Notes</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-sm text-gray-700">{feedback.final_notes}</p>
        </CardContent>
      </Card>

      {/* Action Buttons */}
      <div className="flex gap-3">
        {!feedback.ready_for_submission && roundNumber < 3 && (
          <Button onClick={onNextRound} size="lg" className="flex-1">
            Start Round {roundNumber + 1}
          </Button>
        )}
        {feedback.ready_for_submission && (
          <Button onClick={onFinalize} size="lg" variant="default" className="flex-1">
            <PartyPopper className="mr-2 h-5 w-5" />
            Finalize & Download
          </Button>
        )}
        <Button variant="outline" size="lg">
          Save Progress
        </Button>
      </div>
    </div>
  );
}

4. Create revision upload component in /components/papers/RevisionUpload.tsx:

'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Upload, Loader2 } from 'lucide-react';
import { toast } from 'react-hot-toast';

interface RevisionUploadProps {
  paperId: string;
  roundNumber: number;
  onRevisionSubmitted: () => void;
}

export function RevisionUpload({ paperId, roundNumber, onRevisionSubmitted }: RevisionUploadProps) {
  const [revisionText, setRevisionText] = useState('');
  const [uploading, setUploading] = useState(false);

  const handleSubmit = async () => {
    if (!revisionText.trim()) {
      toast.error('Please paste your revised paper text');
      return;
    }

    setUploading(true);

    try {
      const response = await fetch(`/api/papers/${paperId}/feedback`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          roundNumber,
          currentRevision: revisionText
        })
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error || 'Failed to process revision');
      }

      toast.success(`Round ${roundNumber} feedback generated!`);
      onRevisionSubmitted();
    } catch (error) {
      toast.error(error instanceof Error ? error.message : 'Failed to submit revision');
    } finally {
      setUploading(false);
    }
  };

  return (
    <Card>
      <CardHeader>
        <CardTitle>Upload Your Revision - Round {roundNumber}</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div>
          <p className="text-sm text-gray-600 mb-2">
            Paste the full text of your revised paper below, or upload a new file.
          </p>
          <Textarea
            value={revisionText}
            onChange={(e) => setRevisionText(e.target.value)}
            placeholder="Paste your revised paper text here..."
            rows={15}
            className="font-mono text-sm"
          />
          <p className="text-xs text-gray-500 mt-2">
            {revisionText.length} characters
          </p>
        </div>

        <div className="flex gap-2">
          <Button
            onClick={handleSubmit}
            disabled={!revisionText.trim() || uploading}
            className="flex-1"
          >
            {uploading ? (
              <>
                <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                Processing...
              </>
            ) : (
              <>
                <Upload className="mr-2 h-4 w-4" />
                Submit for Round {roundNumber} Review
              </>
            )}
          </Button>
        </div>
      </CardContent>
    </Card>
  );
}

5. Create progress tracker component in /components/papers/ProgressTracker.tsx:

'use client';

import { Card, CardContent } from '@/components/ui/card';
import { CheckCircle2, Circle, Clock } from 'lucide-react';

interface ProgressTrackerProps {
  currentRound: number;
  rounds: Array<{
    number: number;
    completed: boolean;
    score?: number;
  }>;
}

export function ProgressTracker({ currentRound, rounds }: ProgressTrackerProps) {
  return (
    <Card>
      <CardContent className="pt-6">
        <div className="flex items-center justify-between">
          {[1, 2, 3].map((round) => {
            const roundData = rounds.find(r => r.number === round);
            const completed = roundData?.completed || false;
            const inProgress = currentRound === round;

            return (
              <div key={round} className="flex-1 relative">
                {/* Connector Line */}
                {round < 3 && (
                  <div className={`absolute top-5 left-1/2 w-full h-0.5 ${
                    completed ? 'bg-green-500' : 'bg-gray-300'
                  }`} />
                )}

                {/* Round Indicator */}
                <div className="relative flex flex-col items-center">
                  <div className={`w-10 h-10 rounded-full flex items-center justify-center ${
                    completed ? 'bg-green-500' :
                    inProgress ? 'bg-blue-500' :
                    'bg-gray-300'
                  }`}>
                    {completed ? (
                      <CheckCircle2 className="h-6 w-6 text-white" />
                    ) : inProgress ? (
                      <Clock className="h-6 w-6 text-white" />
                    ) : (
                      <Circle className="h-6 w-6 text-white" />
                    )}
                  </div>
                  <span className="text-sm font-medium mt-2">Round {round}</span>
                  {roundData?.score && (
                    <span className="text-xs text-gray-600 mt-1">
                      {roundData.score}/100
                    </span>
                  )}
                </div>
              </div>
            );
          })}
        </div>
      </CardContent>
    </Card>
  );
}

DELIVERABLES:
- Refinement feedback system (rounds 2-3)
- Cost-efficient GPT-4o-mini usage
- Progress tracking across rounds
- Revision upload flow
- Ready-for-submission indicator

TESTING:
- [ ] Round 2 uses GPT-4o-mini (cheaper)
- [ ] Compares revision to original
- [ ] Tracks resolved vs remaining issues
- [ ] Ready-for-submission flag accurate
- [ ] Cost < $0.05 per round 2-3
```

---

## **DAY 8: CITATION FORMATTER & BIBLIOGRAPHY VALIDATOR**

```
TASK: Build automated citation parser and formatter for Uzbek academic standards

1. Create citation parsing utility in /lib/citation-parser.ts:

export interface Citation {
  type: 'book' | 'article' | 'online' | 'unknown';
  author: string;
  title: string;
  publisher?: string;
  city?: string;
  year?: number;
  pages?: string;
  journal?: string;
  issue?: string;
  url?: string;
  access_date?: string;
  original_text: string;
  is_valid: boolean;
  errors: string[];
}

export const UZBEK_CITATION_PATTERNS = {
  book: /^(.+?)\.\s+(.+?)\.\s*[–-]\s*(.+?):\s*(.+?),\s*(\d{4})\.\s*[–-]\s*(\d+)\s*b\.$/i,
  article: /^(.+?)\.\s+(.+?)\s*\/\/\s*(.+?)\.\s*[–-]\s*(\d{4})\.\s*[–-]\s*№\s*(\d+)\.\s*[–-]\s*B\.\s*(.+?)\.$/i,
  online: /^(.+?)\.\s+(.+?)\.\s*\[.*?resurs\]\s*[–-]\s*URL:\s*(.+?)\s*\((.+?)\)\.$/i
};

export function parseCitation(text: string): Citation {
  const trimmed = text.trim();
  
  // Try to match book pattern
  const bookMatch = trimmed.match(UZBEK_CITATION_PATTERNS.book);
  if (bookMatch) {
    return {
      type: 'book',
      author: bookMatch[1].trim(),
      title: bookMatch[2].trim(),
      publisher: bookMatch[3].trim(),
      city: bookMatch[4].trim(),
      year: parseInt(bookMatch[5]),
      pages: bookMatch[6].trim(),
      original_text: trimmed,
      is_valid: true,
      errors: []
    };
  }

  // Try to match article pattern
  const articleMatch = trimmed.match(UZBEK_CITATION_PATTERNS.article);
  if (articleMatch) {
    return {
      type: 'article',
      author: articleMatch[1].trim(),
      title: articleMatch[2].trim(),
      journal: articleMatch[3].trim(),
      year: parseInt(articleMatch[4]),
      issue: articleMatch[5].trim(),
      pages: articleMatch[6].trim(),
      original_text: trimmed,
      is_valid: true,
      errors: []
    };
  }

  // Try to match online pattern
  const onlineMatch = trimmed.match(UZBEK_CITATION_PATTERNS.online);
  if (onlineMatch) {
    return {
      type: 'online',
      author: onlineMatch[1].trim(),
      title: onlineMatch[2].trim(),
      url: onlineMatch[3].trim(),
      access_date: onlineMatch[4].trim(),
      original_text: trimmed,
      is_valid: true,
      errors: []
    };
  }

  // Unknown/invalid format
  return {
    type: 'unknown',
    author: '',
    title: '',
    original_text: trimmed,
    is_valid: false,
    errors: ['Citation format not recognized']
  };
}

export function formatCitation(citation: Partial<Citation>): string {
  switch (citation.type) {
    case 'book':
      return `${citation.author}. ${citation.title}. – ${citation.publisher}: ${citation.city}, ${citation.year}. – ${citation.pages} b.`;
    
    case 'article':
      return `${citation.author}. ${citation.title} // ${citation.journal}. – ${citation.year}. – № ${citation.issue}. – B. ${citation.pages}.`;
    
    case 'online':
      return `${citation.author}. ${citation.title}. [Elektron resurs] – URL: ${citation.url} (${citation.access_date}).`;
    
    default:
      return citation.original_text || '';
  }
}

export interface BibliographyAnalysis {
  total_citations: number;
  valid_citations: number;
  invalid_citations: number;
  by_type: {
    book: number;
    article: number;
    online: number;
    unknown: number;
  };
  errors: Array<{
    citation_number: number;
    citation_text: string;
    issues: string[];
  }>;
  suggestions: string[];
}

export function analyzeBibliography(bibliographyText: string): BibliographyAnalysis {
  // Split by common delimiters
  const citations = bibliographyText
    .split(/\n+/)
    .map(line => line.trim())
    .filter(line => line.length > 10);

  const analysis: BibliographyAnalysis = {
    total_citations: citations.length,
    valid_citations: 0,
    invalid_citations: 0,
    by_type: { book: 0, article: 0, online: 0, unknown: 0 },
    errors: [],
    suggestions: []
  };

  citations.forEach((citationText, index) => {
    // Remove numbering if present (1., 2., etc.)
    const cleanedText = citationText.replace(/^\d+\.\s*/, '');
    const parsed = parseCitation(cleanedText);

    if (parsed.is_valid) {
      analysis.valid_citations++;
      analysis.by_type[parsed.type]++;
    } else {
      analysis.invalid_citations++;
      analysis.by_type.unknown++;
      analysis.errors.push({
        citation_number: index + 1,
        citation_text: cleanedText.slice(0, 100),
        issues: parsed.errors
      });
    }
  });

  // Generate suggestions
  if (analysis.invalid_citations > 0) {
    analysis.suggestions.push(
      `${analysis.invalid_citations} citations need format correction`
    );
  }

  if (analysis.total_citations < 15) {
    analysis.suggestions.push(
      `Add ${15 - analysis.total_citations} more references (minimum 15 required)`
    );
  }

  if (analysis.by_type.online > analysis.total_citations * 0.3) {
    analysis.suggestions.push(
      'Too many online sources. Include more academic journals and books'
    );
  }

  return analysis;
}

2. Create citation formatter API in /app/api/papers/[id]/citations/format/route.ts:

import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { parseCitation, formatCitation, analyzeBibliography } from '@/lib/citation-parser';

export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const supabase = createRouteHandlerClient({ cookies });
    const { data: { user } } = await supabase.auth.getUser();
    
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const { bibliography_text, auto_fix } = await request.json();

    // Parse all citations
    const citations = bibliography_text
      .split(/\n+/)
      .map((line: string) => line.trim())
      .filter((line: string) => line.length > 10)
      .map((line: string) => {
        const cleaned = line.replace(/^\d+\.\s*/, '');
        return parseCitation(cleaned);
      });

    // Analyze bibliography
    const analysis = analyzeBibliography(bibliography_text);

    // Auto-fix if requested
    let formatted_bibliography = null;
    if (auto_fix) {
      formatted_bibliography = citations
        .map((citation, index) => {
          if (citation.is_valid) {
            return `${index + 1}. ${formatCitation(citation)}`;
          } else {
            return `${index + 1}. ${citation.original_text} // [FORMAT ERROR]`;
          }
        })
        .join('\n');
    }

    // Log usage
    await supabase.from('usage_logs').insert({
      user_id: user.id,
      paper_id: params.id,
      action: 'citation_format_check',
      metadata: { total_citations: analysis.total_citations }
    });

    return NextResponse.json({
      success: true,
      analysis,
      citations,
      formatted_bibliography
    });

  } catch (error) {
    console.error('Citation formatting error:', error);
    return NextResponse.json(
      { error: 'Citation formatting failed' },
      { status: 500 }
    );
  }
}

3. Create citation checker component in /components/papers/CitationChecker.tsx:

'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Textarea } from '@/components/ui/textarea';
import { Badge } from '@/components/ui/badge';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { CheckCircle2, XCircle, AlertCircle, Wand2 } from 'lucide-react';
import { BibliographyAnalysis, Citation } from '@/lib/citation-parser';
import { toast } from 'react-hot-toast';

interface CitationCheckerProps {
  paperId: string;
  initialBibliography?: string;
}

export function CitationChecker({ paperId, initialBibliography = '' }: CitationCheckerProps) {
  const [bibliography, setBibliography] = useState(initialBibliography);
  const [analysis, setAnalysis] = useState<BibliographyAnalysis | null>(null);
  const [citations, setCitations] = useState<Citation[]>([]);
  const [formattedText, setFormattedText] = useState<string | null>(null);
  const [checking, setChecking] = useState(false);

  const handleCheck = async (autoFix: boolean = false) => {
    setChecking(true);

    try {
      const response = await fetch(`/api/papers/${paperId}/citations/format`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          bibliography_text: bibliography,
          auto_fix: autoFix
        })
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error);
      }

      setAnalysis(data.analysis);
      setCitations(data.citations);
      
      if (data.formatted_bibliography) {
        setFormattedText(data.formatted_bibliography);
        toast.success('Citations auto-formatted!');
      } else {
        toast.success('Citation check complete');
      }

    } catch (error) {
      toast.error(error instanceof Error ? error.message : 'Check failed');
    } finally {
      setChecking(false);
    }
  };

  const applyFormatting = () => {
    if (formattedText) {
      setBibliography(formattedText);
      setFormattedText(null);
      toast.success('Formatting applied!');
    }
  };

  return (
    <div className="space-y-6">
      <Card>
        <CardHeader>
          <CardTitle>Bibliography / Foydalanilgan adabiyotlar</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          <div>
            <Textarea
              value={bibliography}
              onChange={(e) => setBibliography(e.target.value)}
              placeholder="Paste your bibliography here... Each citation on a new line."
              rows={15}
              className="font-mono text-sm"
            />
            <p className="text-xs text-gray-500 mt-2">
              {bibliography.split('\n').filter(l => l.trim()).length} citations detected
            </p>
          </div>

          <div className="flex gap-2">
            <Button 
              onClick={() => handleCheck(false)}
              disabled={!bibliography.trim() || checking}
              variant="outline"
            >
              Check Citations
            </Button>
            <Button 
              onClick={() => handleCheck(true)}
              disabled={!bibliography.trim() || checking}
            >
              <Wand2 className="mr-2 h-4 w-4" />
              Auto-Fix Formatting
            </Button>
          </div>
        </CardContent>
      </Card>

      {/* Analysis Results */}
      {analysis && (
        <>
          <Card>
            <CardHeader>
              <CardTitle>Citation Analysis</CardTitle>
            </CardHeader>
            <CardContent>
              <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
                <div className="text-center">
                  <div className="text-2xl font-bold">{analysis.total_citations}</div>
                  <div className="text-xs text-gray-500">Total Citations</div>
                </div>
                <div className="text-center">
                  <div className="text-2xl font-bold text-green-600">{analysis.valid_citations}</div>
                  <div className="text-xs text-gray-500">Valid Format</div>
                </div>
                <div className="text-center">
                  <div className="text-2xl font-bold text-red-600">{analysis.invalid_citations}</div>
                  <div className="text-xs text-gray-500">Need Fixing</div>
                </div>
                <div className="text-center">
                  <div className="text-2xl font-bold text-blue-600">
                    {Math.round((analysis.valid_citations / analysis.total_citations) * 100)}%
                  </div>
                  <div className="text-xs text-gray-500">Compliance</div>
                </div>
              </div>

              <div className="mt-4 flex flex-wrap gap-2">
                <Badge variant="outline">
                  📚 Books: {analysis.by_type.book}
                </Badge>
                <Badge variant="outline">
                  📄 Articles: {analysis.by_type.article}
                </Badge>
                <Badge variant="outline">
                  🌐 Online: {analysis.by_type.online}
                </Badge>
                {analysis.by_type.unknown > 0 && (
                  <Badge variant="destructive">
                    ❓ Unknown: {analysis.by_type.unknown}
                  </Badge>
                )}
              </div>
            </CardContent>
          </Card>

          {/* Errors */}
          {analysis.errors.length > 0 && (
            <Card>
              <CardHeader>
                <CardTitle className="text-red-600">Formatting Errors</CardTitle>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  {analysis.errors.map((error, i) => (
                    <Alert key={i} variant="destructive">
                      <XCircle className="h-4 w-4" />
                      <AlertDescription>
                        <span className="font-medium">Citation {error.citation_number}:</span>
                        <p className="text-xs mt-1">{error.citation_text}</p>
                        <ul className="text-xs mt-2 space-y-1">
                          {error.issues.map((issue, j) => (
                            <li key={j}>• {issue}</li>
                          ))}
                        </ul>
                      </AlertDescription>
                    </Alert>
                  ))}
                </div>
              </CardContent>
            </Card>
          )}

          {/* Suggestions */}
          {analysis.suggestions.length > 0 && (
            <Card>
              <CardHeader>
                <CardTitle>💡 Suggestions</CardTitle>
              </CardHeader>
              <CardContent>
                <ul className="space-y-2">
                  {analysis.suggestions.map((suggestion, i) => (
                    <li key={i} className="flex gap-2 text-sm">
                      <AlertCircle className="h-4 w-4 text-yellow-600 flex-shrink-0 mt-0.5" />
                      <span>{suggestion}</span>
                    </li>
                  ))}
                </ul>
              </CardContent>
            </Card>
          )}

          {/* Formatted Output */}
          {formattedText && (
            <Card>
              <CardHeader>
                <CardTitle className="text-green-600">✨ Auto-Formatted Citations</CardTitle>
              </CardHeader>
              <CardContent className="space-y-4">
                <Textarea
                  value={formattedText}
                  readOnly
                  rows={15}
                  className="font-mono text-sm bg-green-50"
                />
                <Button onClick={applyFormatting}>
                  <CheckCircle2 className="mr-2 h-4 w-4" />
                  Apply This Formatting
                </Button>
              </CardContent>
            </Card>
          )}
        </>
      )}
    </div>
  );
}

4. Add citation format guide in /components/papers/CitationGuide.tsx:

'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';

export function CitationGuide() {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Uzbek Citation Format Guide</CardTitle>
      </CardHeader>
      <CardContent>
        <Tabs defaultValue="book">
          <TabsList className="grid w-full grid-cols-3">
            <TabsTrigger value="book">Book</TabsTrigger>
            <TabsTrigger value="article">Article</TabsTrigger>
            <TabsTrigger value="online">Online</TabsTrigger>
          </TabsList>

          <TabsContent value="book" className="space-y-2">
            <div className="p-3 bg-gray-50 rounded">
              <p className="text-xs font-medium text-gray-600 mb-1">Format:</p>
              <p className="font-mono text-sm">
                Author. Title. – Publisher: City, Year. – Pages b.
              </p>
            </div>
            <div className="p-3 bg-blue-50 rounded">
              <p className="text-xs font-medium text-blue-600 mb-1">Example:</p>
              <p className="font-mono text-xs">
                Karimov I.A. O'zbekiston XXI asr bo'sag'asida. – T.: O'zbekiston, 1997. – 320 b.
              </p>
            </div>
          </TabsContent>

          <TabsContent value="article" className="space-y-2">
            <div className="p-3 bg-gray-50 rounded">
              <p className="text-xs font-medium text-gray-600 mb-1">Format:</p>
              <p className="font-mono text-sm">
                Author. Title // Journal. – Year. – № Issue. – B. Pages.
              </p>
            </div>
            <div className="p-3 bg-blue-50 rounded">
              <p className="text-xs font-medium text-blue-600 mb-1">Example:</p>
              <p className="font-mono text-xs">
                Ergashev B. Milliy g'oya asoslari // Tafakkur. – 2020. – № 2. – B. 15-23.
              </p>
            </div>
          </TabsContent>

          <TabsContent value="online" className="space-y-2">
            <div className="p-3 bg-gray-50 rounded">
              <p className="text-xs font-medium text-gray-600 mb-1">Format:</p>
              <p className="font-mono text-sm">
                Author. Title. [Elektron resurs] – URL: url (access_date).
              </p>
            </div>
            <div className="p-3 bg-blue-50 rounded">
              <p className="text-xs font-medium text-blue-600 mb-1">Example:</p>
              <p className="font-mono text-xs">
                Rahmonov S. Digital iqtisodiyot. [Elektron resurs] – URL: https://example.uz (20.01.2024).
              </p>
            </div>
          </TabsContent>
        </Tabs>
      </CardContent>
    </Card>
  );
}

DELIVERABLES:
- Citation parser for Uzbek formats
- Auto-formatter for citations
- Bibliography validator
- Format compliance checker
- Citation type detector

TESTING:
- [ ] Correctly parses book citations
- [ ] Correctly parses article citations
- [ ] Correctly parses online citations
- [ ] Auto-formats invalid citations
- [ ] Detects minimum 15 references
- [ ] Shows format errors clearly
```

---

## **DAY 9: USER DASHBOARD & USAGE TRACKING**

```
TASK: Build comprehensive user dashboard with usage stats and paper management

1. Create dashboard page in /app/dashboard/page.tsx:

import { createServerComponentClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { UsageCard } from '@/components/dashboard/UsageCard';
import { PapersList } from '@/components/dashboard/PapersList';
import { StatsOverview } from '@/components/dashboard/StatsOverview';
import { PaperUpload } from '@/components/papers/PaperUpload';

export default async function DashboardPage() {
  const supabase = createServerComponentClient({ cookies });
  
  const { data: { user } } = await supabase.auth.getUser();
  
  if (!user) {
    redirect('/login');
  }

  // Get subscription info
  const { data: subscription } = await supabase
    .from('subscriptions')
    .select('*')
    .eq('user_id', user.id)
    .single();

  // Get papers
  const { data: papers } = await supabase
    .from('papers')
    .select(`
      *,
      feedback_rounds(*)
    `)
    .eq('user_id', user.id)
    .order('created_at', { ascending: false });

  // Calculate stats
  const totalPapers = papers?.length || 0;
  const completedPapers = papers?.filter(p => p.status === 'completed').length || 0;
  const avgScore = papers?.reduce((acc, paper) => {
    const latestFeedback = paper.feedback_rounds
      ?.sort((a: any, b: any) => b.round_number - a.round_number)[0];
    return acc + (latestFeedback?.content_feedback?.overall_assessment?.score || 0);
  }, 0) / (totalPapers || 1);

  return (
    <div className="container mx-auto py-8 space-y-8">
      <div>
        <h1 className="text-3xl font-bold">Dashboard</h1>
        <p className="text-gray-600">Welcome back, {user.email}</p>
      </div>

      {/* Usage Card */}
      <UsageCard subscription={subscription} />

      {/* Stats Overview */}
      <StatsOverview
        totalPapers={totalPapers}
        completedPapers={completedPapers}
        avgScore={Math.round(avgScore)}
      />

      {/* Quick Upload */}
      <div>
        <h2 className="text-xl font-semibold mb-4">Upload New Paper</h2>
        <PaperUpload />
      </div>

      {/* Papers List */}
      <div>
        <h2 className="text-xl font-semibold mb-4">Your Papers</h2>
        <PapersList papers={papers || []} />
      </div>
    </div>
  );
}

2. Create usage card component in /components/dashboard/UsageCard.tsx:

'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Progress } from '@/components/ui/progress';
import { Badge } from '@/components/ui/badge';
import { FileText, CreditCard, Calendar } from 'lucide-react';
import { useRouter } from 'next/navigation';
import { format } from 'date-fns';

interface UsageCardProps {
  subscription: {
    plan_type: string;
    papers_remaining: number;
    total_papers_allowed: number;
    expires_at: string;
  } | null;
}

export function UsageCard({ subscription }: UsageCardProps) {
  const router = useRouter();
  
  if (!subscription) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>No Active Subscription</CardTitle>
        </CardHeader>
        <CardContent>
          <Button onClick={() => router.push('/pricing')}>
            Subscribe Now
          </Button>
        </CardContent>
      </Card>
    );
  }

  const usagePercent = 
    ((subscription.total_papers_allowed - subscription.papers_remaining) / 
    subscription.total_papers_allowed) * 100;

  return (
    <Card>
      <CardHeader>
        <div className="flex items-center justify-between">
          <CardTitle>Your Subscription</CardTitle>
          <Badge variant={subscription.plan_type === 'paid' ? 'default' : 'secondary'}>
            {subscription.plan_type === 'paid' ? 'Premium' : 'Free'}
          </Badge>
        </div>
      </CardHeader>
      <CardContent className="space-y-6">
        {/* Papers Remaining */}
        <div className="space-y-2">
          <div className="flex items-center justify-between">
            <div className="flex items-center gap-2">
              <FileText className="h-4 w-4 text-gray-500" />
              <span className="text-sm font-medium">Papers Remaining</span>
            </div>
            <span className="text-2xl font-bold">
              {subscription.papers_remaining} / {subscription.total_papers_allowed}
            </span>
          </div>
          <Progress value={100 - usagePercent} />
          <p className="text-xs text-gray-500">
            {subscription.papers_remaining === 0 ? (
              <span className="text-red-600 font-medium">
                No papers remaining. Upgrade to continue.
              </span>
            ) : (
              `${subscription.papers_remaining} papers left this month`
            )}
          </p>
        </div>

        {/* Expiry Date */}
        <div className="flex items-center justify-between p-3 bg-gray-50 rounded-lg">
          <div className="flex items-center gap-2">
            <Calendar className="h-4 w-4 text-gray-500" />
            <span className="text-sm">Resets on</span>
          </div>
          <span className="text-sm font-medium">
            {format(new Date(subscription.expires_at), 'MMM dd, yyyy')}
          </span>
        </div>

        {/* Action Buttons */}
        <div className="flex gap-2">
          {subscription.papers_remaining === 0 && (
            <Button className="flex-1" onClick={() => router.push('/pricing')}>
              <CreditCard className="mr-2 h-4 w-4" />
              Buy More Papers
            </Button>
          )}
          {subscription.plan_type === 'free' && (
            <Button variant="outline" className="flex-1" onClick={() => router.push('/pricing')}>
              Upgrade to Premium
            </Button>
          )}
        </div>
      </CardContent>
    </Card>
  );
}

3. Create stats overview component in /components/dashboard/StatsOverview.tsx:

'use client';

import { Card, CardContent } from '@/components/ui/card';
import { FileText, CheckCircle2, TrendingUp, Award } from 'lucide-react';

interface StatsOverviewProps {
  totalPapers: number;
  completedPapers: number;
  avgScore: number;
}

export function StatsOverview({ totalPapers, completedPapers, avgScore }: StatsOverviewProps) {
  const completionRate = totalPapers > 0 
    ? Math.round((completedPapers / totalPapers) * 100) 
    : 0;

  const stats = [
    {
      label: 'Total Papers',
      value: totalPapers,
      icon: FileText,
      color: 'text-blue-600',
      bgColor: 'bg-blue-50'
    },
    {
      label: 'Completed',
      value: completedPapers,
      icon: CheckCircle2,
      color: 'text-green-600',
      bgColor: 'bg-green-50'
    },
    {
      label: 'Completion Rate',
      value: `${completionRate}%`,
      icon: TrendingUp,
      color: 'text-purple-600',
      bgColor: 'bg-purple-50'
    },
    {
      label: 'Avg Score',
      value: `${avgScore}/100`,
      icon: Award,
      color: 'text-yellow-600',
      bgColor: 'bg-yellow-50'
    }
  ];

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
      {stats.map((stat) => {
        const Icon = stat.icon;
        return (
          <Card key={stat.label}>
            <CardContent className="pt-6">
              <div className="flex items-center gap-4">
                <div className={`p-3 rounded-lg ${stat.bgColor}`}>
                  <Icon className={`h-6 w-6 ${stat.color}`} />
                </div>
                <div>
                  <p className="text-sm text-gray-600">{stat.label}</p>
                  <p className="text-2xl font-bold">{stat.value}</p>
                </div>
              </div>
            </CardContent>
          </Card>
        );
      })}
    </div>
  );
}

4. Create papers list component in /components/dashboard/PapersList.tsx:

'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { FileText, Clock, CheckCircle2, AlertCircle, Download, Eye } from 'lucide-react';
import { useRouter } from 'next/navigation';
import { formatDistanceToNow } from 'date-fns';

interface PapersListProps {
  papers: Array<{
    id: string;
    title: string;
    status: string;
    page_count: number;
    language: string;
    created_at: string;
    feedback_rounds: any[];
  }>;
}

export function PapersList({ papers }: PapersListProps) {
  const router = useRouter();

  if (papers.length === 0) {
    return (
      <Card>
        <CardContent className="py-12 text-center">
          <FileText className="h-12 w-12 mx-auto text-gray-400 mb-4" />
          <h3 className="text-lg font-medium mb-2">No papers yet</h3>
          <p className="text-sm text-gray-600 mb-4">
            Upload your first paper to get started with AI feedback
          </p>
        </CardContent>
      </Card>
    );
  }

  const getStatusIcon = (status: string) => {
    switch (status) {
      case 'completed':
        return <CheckCircle2 className="h-4 w-4 text-green-600" />;
      case 'processing':
        return <Clock className="h-4 w-4 text-blue-600 animate-spin" />;
      case 'failed':
        return <AlertCircle className="h-4 w-4 text-red-600" />;
      default:
        return <FileText className="h-4 w-4 text-gray-600" />;
    }
  };

  const getStatusBadge = (status: string) => {
    switch (status) {
      case 'completed':
        return <Badge variant="default">Completed</Badge>;
      case 'processing':
        return <Badge variant="secondary">Processing</Badge>;
      case 'failed':
        return <Badge variant="destructive">Failed</Badge>;
      default:
        return <Badge variant="outline">Uploaded</Badge>;
    }
  };

  return (
    <div className="space-y-4">
      {papers.map((paper) => {
        const latestFeedback = paper.feedback_rounds
          ?.sort((a, b) => b.round_number - a.round_number)[0];
        const score = latestFeedback?.content_feedback?.overall_assessment?.score;

        return (
          <Card key={paper.id}>
            <CardHeader>
              <div className="flex items-start justify-between">
                <div className="flex items-start gap-3 flex-1">
                  {getStatusIcon(paper.status)}
                  <div className="flex-1">
                    <CardTitle className="text-lg">{paper.title}</CardTitle>
                    <div className="flex flex-wrap gap-2 mt-2">
                      {getStatusBadge(paper.status)}
                      <Badge variant="outline">{paper.page_count} pages</Badge>
                      <Badge variant="outline">{paper.language}</Badge>
                      {score && (
                        <Badge variant={score >= 80 ? 'default' : 'secondary'}>
                          Score: {score}/100
                        </Badge>
                      )}
                      {paper.feedback_rounds.length > 0 && (
                        <Badge variant="outline">
                          {paper.feedback_rounds.length} round{paper.feedback_rounds.length > 1 ? 's' : ''}
                        </Badge>
                      )}
                    </div>
                  </div>
                </div>
                <div className="text-sm text-gray-500">
                  {formatDistanceToNow(new Date(paper.created_at), { addSuffix: true })}
                </div>
              </div>
            </CardHeader>
            <CardContent>
              <div className="flex gap-2">
                <Button
                  size="sm"
                  onClick={() => router.push(`/papers/${paper.id}`)}
                >
                  <Eye className="mr-2 h-4 w-4" />
                  View Details
                </Button>
                {paper.status === 'completed' && (
                  <Button
                    size="sm"
                    variant="outline"
                    onClick={() => window.open(`/api/papers/${paper.id}/export?format=docx`, '_blank')}
                  >
                    <Download className="mr-2 h-4 w-4" />
                    Download
                  </Button>
                )}
                {paper.status === 'uploaded' && (
                  <Button
                    size="sm"
                    variant="default"
                    onClick={() => router.push(`/papers/${paper.id}`)}
                  >
                    Start Review
                  </Button>
                )}
              </div>
            </CardContent>
          </Card>
        );
      })}
    </div>
  );
}

5. Create usage tracking utility in /lib/analytics.ts:

import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

export async function trackUsage(
  userId: string,
  action: string,
  metadata?: any
) {
  try {
    await supabase.from('usage_logs').insert({
      user_id: userId,
      action,
      metadata,
      created_at: new Date().toISOString()
    });
  } catch (error) {
    console.error('Failed to track usage:', error);
  }
}

export async function getUserStats(userId: string) {
  // Get papers stats
  const { data: papers } = await supabase
    .from('papers')
    .select('id, status, created_at, feedback_rounds(content_feedback)')
    .eq('user_id', userId);

  // Get usage logs
  const { data: logs } = await supabase
    .from('usage_logs')
    .select('*')
    .eq('user_id', userId)
    .order('created_at', { ascending: false })
    .limit(100);

  // Calculate stats
  const totalPapers = papers?.length || 0;
  const completedPapers = papers?.filter(p => p.status === 'completed').length || 0;
  
  const scores = papers
    ?.map(p => p.feedback_rounds?.[0]?.content_feedback?.overall_assessment?.score)
    .filter(Boolean) || [];
  const avgScore = scores.length > 0 
    ? Math.round(scores.reduce((a, b) => a + b, 0) / scores.length)
    : 0;

  const totalTokens = logs?.reduce((sum, log) => sum + (log.tokens_used || 0), 0) || 0;
  const totalCost = logs?.reduce((sum, log) => sum + (log.cost_usd || 0), 0) || 0;

  return {
    totalPapers,
    completedPapers,
    avgScore,
    totalTokens,
    totalCost,
    recentActivity: logs?.slice(0, 10)
  };
}

DELIVERABLES:
- Complete user dashboard
- Usage tracking system
- Papers management interface
- Statistics visualization
- Subscription status display

TESTING:
- [ ] Dashboard loads correctly
- [ ] Shows accurate paper counts
- [ ] Displays subscription status
- [ ] Papers list functional
- [ ] Usage stats accurate
```

---

## **DAY 10: TESTING, DEPLOYMENT & USER ONBOARDING**

```
TASK: Final testing, production deployment, and onboard 20 university students

1. Create comprehensive testing checklist in /docs/TESTING.md:

# Testing Checklist

## Unit Tests
- [ ] PDF text extraction works (Uzbek, Russian, English)
- [ ] Citation parser detects all formats
- [ ] Cost calculation accurate
- [ ] Token counting correct

## API Tests
### Upload
- [ ] Can upload PDF < 50MB
- [ ] Can upload DOCX
- [ ] Rejects files > 50MB
- [ ] Rejects invalid file types
- [ ] Properly extracts text

### Feedback Generation
- [ ] Round 1 generates content + structure feedback
- [ ] Round 2-3 use GPT-4o-mini (cheaper)
- [ ] Feedback format is valid JSON
- [ ] Scores are 0-100
- [ ] Feedback in correct language

### Citations
- [ ] Detects book citations
- [ ] Detects article citations
- [ ] Detects online citations
- [ ] Auto-formats correctly
- [ ] Identifies errors

### Export
- [ ] Export DOCX works
- [ ] Export PDF works
- [ ] Files download correctly

## UI Tests
- [ ] Dashboard loads
- [ ] Paper upload works
- [ ] Feedback displays correctly
- [ ] Comparison view functional
- [ ] Mobile responsive

## Integration Tests
- [ ] Full workflow: Upload → R1 → R2 → R3 → Export
- [ ] Subscription limits enforced
- [ ] Usage tracking accurate
- [ ] Cost tracking correct

## Performance Tests
- [ ] API responds < 2s for simple requests
- [ ] Feedback generation < 30s
- [ ] Handles 10 concurrent uploads
- [ ] Database queries optimized

## Security Tests
- [ ] RLS policies enforce user isolation
- [ ] Cannot access other users' papers
- [ ] API requires authentication
- [ ] File uploads validated

2. Create deployment script in /scripts/deploy.sh:

#!/bin/bash

echo "🚀 Deploying IlmAI to Production..."

# Check environment variables
if [ -z "$OPENAI_API_KEY" ]; then
  echo "❌ OPENAI_API_KEY not set"
  exit 1
fi

if [ -z "$NEXT_PUBLIC_SUPABASE_URL" ]; then
  echo "❌ NEXT_PUBLIC_SUPABASE_URL not set"
  exit 1
fi

# Run tests
echo "🧪 Running tests..."
npm run test

if [ $? -ne 0 ]; then
  echo "❌ Tests failed"
  exit 1
fi

# Build
echo "📦 Building application..."
npm run build

if [ $? -ne 0 ]; then
  echo "❌ Build failed"
  exit 1
fi

# Deploy to Vercel
echo "☁️  Deploying to Vercel..."
vercel --prod

if [ $? -ne 0 ]; then
  echo "❌ Deployment failed"
  exit 1
fi

# Run database migrations
echo "🗄️  Running database migrations..."
supabase db push --db-url $PRODUCTION_DB_URL

if [ $? -ne 0 ]; then
  echo "❌ Database migration failed"
  exit 1
fi

echo "✅ Deployment complete!"
echo "🌐 App URL: https://ilmai.uz"

3. Create user onboarding email template in /emails/onboarding.html:

<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    body {
      font-family: Arial, sans-serif;
      line-height: 1.6;
      color: #333;
    }
    .container {
      max-width: 600px;
      margin: 0 auto;
      padding: 20px;
    }
    .header {
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: white;
      padding: 30px;
      text-align: center;
      border-radius: 10px 10px 0 0;
    }
    .content {
      background: white;
      padding: 30px;
      border: 1px solid #ddd;
      border-top: none;
    }
    .button {
      display: inline-block;
      padding: 12px 30px;
      background: #667eea;
      color: white;
      text-decoration: none;
      border-radius: 5px;
      margin: 20px 0;
    }
    .feature {
      margin: 15px 0;
      padding-left: 25px;
    }
    .footer {
      text-align: center;
      padding: 20px;
      color: #666;
      font-size: 12px;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>🎓 Welcome to IlmAI!</h1>
      <p>Your AI-Powered Academic Writing Assistant</p>
    </div>
    
    <div class="content">
      <h2>Hi {{name}},</h2>
      
      <p>Congratulations! You're one of 20 students selected for our exclusive beta test.</p>
      
      <p><strong>What you get:</strong></p>
      <div class="feature">✅ FREE access for 1 month</div>
      <div class="feature">✅ Review up to 5 papers</div>
      <div class="feature">✅ AI feedback in minutes</div>
      <div class="feature">✅ Structure compliance checking</div>
      <div class="feature">✅ Citation formatting</div>
      
      <h3>Getting Started (3 Easy Steps):</h3>
      <ol>
        <li><strong>Create your account</strong> using code: <code>TEST20</code></li>
        <li><strong>Upload your paper</strong> (PDF or DOCX)</li>
        <li><strong>Get instant feedback</strong> on content & structure</li>
      </ol>
      
      <center>
        <a href="https://ilmai.uz/signup?code=TEST20" class="button">
          Create Account Now
        </a>
      </center>
      
      <h3>Need Help?</h3>
      <p>Watch our 2-minute tutorial: <a href="https://ilmai.uz/tutorial">ilmai.uz/tutorial</a></p>
      <p>Questions? Reply to this email anytime.</p>
      
      <p><strong>Important:</strong> We need your honest feedback to improve IlmAI. After using the tool, you'll receive a short survey.</p>
      
      <p>Good luck with your research! 📚</p>
      
      <p>Best regards,<br>
      <strong>The IlmAI Team</strong></p>
    </div>
    
    <div class="footer">
      <p>IlmAI - AI for Uzbek Academic Excellence</p>
      <p>You received this email because you signed up for our beta program.</p>
    </div>
  </div>
</body>
</html>

4. Create feedback survey in /app/feedback/page.tsx:

'use client';

import { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { RadioGroup, RadioGroupItem } from '@/components/ui/radio-group';
import { Label } from '@/components/ui/label';
import { Checkbox } from '@/components/ui/checkbox';
import { toast } from 'react-hot-toast';

export default function FeedbackPage() {
  const [ratings, setRatings] = useState({
    quality: '',
    ease: '',
    speed: '',
    usefulness: ''
  });
  const [feedback, setFeedback] = useState({
    liked: '',
    improve: '',
    wouldPay: false
  });
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async () => {
    // Validate
    if (!ratings.quality || !ratings.ease || !ratings.speed || !ratings.usefulness) {
      toast.error('Please answer all rating questions');
      return;
    }

    if (!feedback.liked || !feedback.improve) {
      toast.error('Please answer all text questions');
      return;
    }

    setSubmitting(true);

    try {
      const response = await fetch('/api/feedback', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ ratings, feedback })
      });

      if (!response.ok) throw new Error('Failed to submit');

      toast.success('Thank you for your feedback! 🎉');
      
      // Redirect to dashboard
      setTimeout(() => {
        window.location.href = '/dashboard';
      }, 2000);

    } catch (error) {
      toast.error('Failed to submit feedback');
    } finally {
      setSubmitting(false);
    }
  };

  const RatingQuestion = ({ 
    question, 
    field 
  }: { 
    question: string; 
    field: keyof typeof ratings;
  }) => (
    <div className="space-y-2">
      <Label className="text-base font-medium">{question}</Label>
      <RadioGroup
        value={ratings[field]}
        onValueChange={(value) => setRatings({ ...ratings, [field]: value })}
      >
        <div className="flex gap-4">
          {[1, 2, 3, 4, 5].map((value) => (
            <div key={value} className="flex items-center space-x-2">
              <RadioGroupItem value={value.toString()} id={`${field}-${value}`} />
              <Label htmlFor={`${field}-${value}`}>{value}</Label>
            </div>
          ))}
        </div>
      </RadioGroup>
      <p className="text-xs text-gray-500">1 = Very Poor, 5 = Excellent</p>
    </div>
  );

  return (
    <div className="container mx-auto py-8 max-w-3xl">
      <Card>
        <CardHeader>
          <CardTitle className="text-2xl">📝 Your Feedback Matters!</CardTitle>
          <p className="text-gray-600">
            Help us improve IlmAI by sharing your experience
          </p>
        </CardHeader>
        <CardContent className="space-y-8">
          {/* Ratings */}
          <div className="space-y-6">
            <h3 className="text-lg font-semibold">Rate Your Experience</h3>
            
            <RatingQuestion
              question="How would you rate the quality of feedback?"
              field="quality"
            />
            
            <RatingQuestion
              question="How easy was it to use the platform?"
              field="ease"
            />
            
            <RatingQuestion
              question="How satisfied are you with the speed of feedback?"
              field="speed"
            />
            
            <RatingQuestion
              question="How useful was the feedback for improving your paper?"
              field="usefulness"
            />
          </div>

          {/* Open-ended Questions */}
          <div className="space-y-4">
            <h3 className="text-lg font-semibold">Tell Us More</h3>
            
            <div className="space-y-2">
              <Label>What did you like most about IlmAI?</Label>
              <Textarea
                value={feedback.liked}
                onChange={(e) => setFeedback({ ...feedback, liked: e.target.value })}
                placeholder="Be specific..."
                rows={4}
                required
              />
            </div>

            <div className="space-y-2">
              <Label>What needs improvement?</Label>
              <Textarea
                value={feedback.improve}
                onChange={(e) => setFeedback({ ...feedback, improve: e.target.value })}
                placeholder="Be honest - we want to improve!"
                rows={4}
                required
              />
            </div>

            <div className="flex items-center space-x-2">
              <Checkbox
                id="would-pay"
                checked={feedback.wouldPay}
                onCheckedChange={(checked) => 
                  setFeedback({ ...feedback, wouldPay: checked as boolean })
                }
              />
              <Label htmlFor="would-pay" className="font-normal">
                I would pay $10/month for this service
              </Label>
            </div>
          </div>

          <Button
            onClick={handleSubmit}
            disabled={submitting}
            className="w-full"
            size="lg"
          >
            {submitting ? 'Submitting...' : 'Submit Feedback'}
          </Button>
        </CardContent>
      </Card>
    </div>
  );
}

5. Create monitoring dashboard in /app/admin/monitoring/page.tsx:

// Admin-only page to monitor system health

import { createServerComponentClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

export default async function MonitoringPage() {
  const supabase = createServerComponentClient({ cookies });
  
  // Check if admin (add admin email to env)
  const { data: { user } } = await supabase.auth.getUser();
  if (!user || user.email !== process.env.ADMIN_EMAIL) {
    redirect('/dashboard');
  }

  // Get system stats
  const { data: papers } = await supabase
    .from('papers')
    .select('id, status, created_at');

  const { data: logs } = await supabase
    .from('usage_logs')
    .select('cost_usd, tokens_used, created_at')
    .gte('created_at', new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString());

  const totalCost24h = logs?.reduce((sum, log) => sum + (log.cost_usd || 0), 0) || 0;
  const totalTokens24h = logs?.reduce((sum, log) => sum + (log.tokens_used || 0), 0) || 0;

  const statusCounts = {
    uploaded: papers?.filter(p => p.status === 'uploaded').length || 0,
    processing: papers?.filter(p => p.status === 'processing').length || 0,
    completed: papers?.filter(p => p.status === 'completed').length || 0,
    failed: papers?.filter(p => p.status === 'failed').length || 0
  };

  return (
    <div className="container mx-auto py-8 space-y-6">
      <h1 className="text-3xl font-bold">System Monitoring</h1>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <Card>
          <CardHeader>
            <CardTitle>Cost (24h)</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-3xl font-bold">${totalCost24h.toFixed(2)}</p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader>
            <CardTitle>Tokens (24h)</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-3xl font-bold">{(totalTokens24h / 1000).toFixed(1)}K</p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader>
            <CardTitle>Total Papers</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-3xl font-bold">{papers?.length || 0}</p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader>
            <CardTitle>Failed Papers</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-3xl font-bold text-red-600">{statusCounts.failed}</p>
          </CardContent>
        </Card>
      </div>

      {/* Status Breakdown */}
      <Card>
        <CardHeader>
          <CardTitle>Paper Status Breakdown</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-2">
            <div className="flex justify-between">
              <span>Uploaded:</span>
              <span className="font-bold">{statusCounts.uploaded}</span>
            </div>
            <div className="flex justify-between">
              <span>Processing:</span>
              <span className="font-bold">{statusCounts.processing}</span>
            </div>
            <div className="flex justify-between">
              <span>Completed:</span>
              <span className="font-bold text-green-600">{statusCounts.completed}</span>
            </div>
            <div className="flex justify-between">
              <span>Failed:</span>
              <span className="font-bold text-red-600">{statusCounts.failed}</span>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}

6. Final deployment checklist:

# Production Deployment Checklist

## Pre-Deployment
- [ ] All environment variables set in Vercel
- [ ] Database migrations tested
- [ ] OpenAI API key funded
- [ ] Supabase production database ready
- [ ] All tests passing
- [ ] Build successful locally

## Deployment
- [ ] Deploy to Vercel production
- [ ] Run database migrations
- [ ] Verify app loads
- [ ] Test critical user flows
- [ ] Check error monitoring

## Post-Deployment
- [ ] Create 20 test accounts with code TEST20
- [ ] Send onboarding emails
- [ ] Monitor error logs
- [ ] Watch API costs
- [ ] Collect feedback

## Success Metrics (Track Daily)
- [ ] Papers uploaded: Target 40+ in 10 days
- [ ] Completion rate: Target 80%
- [ ] Avg satisfaction: Target 4.0+/5.0
- [ ] Error rate: Target <5%
- [ ] Cost per paper: Target <$0.20

DELIVERABLES:
- Comprehensive testing suite
- Deployment automation
- User onboarding flow
- Feedback collection system
- Monitoring dashboard

FINAL VERIFICATION:
- [ ] All 10 days completed
- [ ] MVP fully functional
- [ ] Ready for 20 test users
- [ ] Monitoring in place
- [ ] Support process ready
```

---

## **🎉 10-DAY ROADMAP COMPLETE!**

You now have:
1. ✅ Complete technical roadmap for PM/documentation
2. ✅ All 10 days of Cursor prompts for junior devs
3. ✅ Testing & deployment procedures
4. ✅ User onboarding process

**Next Steps:**
1. **Start Day 1 immediately** - Set up project structure
2. **Assign days to your team** - Junior devs can work in parallel on different components
3. **Daily standups** - Review progress, blockers
4. **Day 8 onwards** - Start recruiting 20 test students
5. **Day 10** - Launch and gather feedback

**Key Success Factors:**
- Stick to the timeline - don't add features mid-sprint
- Test each component as you build
- Monitor costs daily
- Gather user feedback actively

Good luck with your launch! 🚀
