## **📋 CO-FOUNDER MANUAL CHECKLIST**

### **Day 1: Data Organization**
- [ ] Create Google Drive folder: "IlmAI_Papers"
- [ ] Upload all 20-40 PDFs with naming: `YYYY_AuthorLastName_ShortTitle.pdf`
- [ ] Create Google Sheet: "Papers_Database" with columns:
  - filename
  - title_uzbek
  - title_english (if available)
  - authors
  - year
  - university
  - keywords (5-10 words)
  - language (uzbek/russian/english)
- [ ] Fill sheet for all papers (can use AI to help extract info)
- [ ] Share both Drive folder + Sheet with technical team

### **Day 2: Quality Check**
- [ ] Review first 5 processed papers in system
- [ ] Check if summaries are accurate
- [ ] Flag any OCR errors or garbled text
- [ ] Find 5 more papers in same domain (agriculture/education/whatever you picked)

### **Day 3: Bulk Processing**
- [ ] Run processing script on remaining papers (engineer will set this up)
- [ ] Test search with 10 different queries in Uzbek/Russian/English:
  - General topics
  - Author names  
  - Specific keywords
- [ ] Document what works/doesn't work

### **Day 4: Demo Content Prep**
- [ ] Write 3 demo scenarios with expected results:
  - "Show me papers about [specific topic]"
  - "Who has researched [domain]?"
  - "Find papers similar to [paper title]"
- [ ] Collect 10 more high-quality papers if dataset feels thin
- [ ] Take screenshots of best search results

### **Day 5-6: Demo Script Writing**
- [ ] Write 2-minute demo script:
  - Problem statement (30 sec)
  - Live demo (60 sec)
  - Future vision (30 sec)
- [ ] Practice demo delivery 5+ times
- [ ] Prepare answers to obvious VC questions:
  - "How many papers will you have at launch?"
  - "How do you ensure quality?"
  - "What's your data collection strategy?"
  - "Why can't universities do this themselves?"

### **Day 7: Final Testing**
- [ ] Test demo on different devices (mobile, tablet, laptop)
- [ ] Check all PDF download links work
- [ ] Have 2 backup demo scenarios ready
- [ ] Print/prepare pitch deck if needed

---

## **💻 CURSOR PROMPTS (Copy-Paste for Junior Devs)**

### **DAY 1: Project Setup**

```
You are building "IlmAI" - a research paper search engine for Uzbek academic papers using RAG (Retrieval Augmented Generation).

TECH STACK:
- Next.js 14 (App Router)
- TypeScript
- Tailwind CSS
- Supabase (PostgreSQL + Vector)
- OpenAI API (embeddings + GPT-4-mini)

TASK: Set up the base project structure

1. Create new Next.js 14 project with TypeScript and Tailwind
2. Install dependencies:
   - @supabase/supabase-js
   - openai
   - react-icons

3. Create folder structure:
   /app
     /api
       /search
       /process-pdf
     /dashboard
     page.tsx (home with search)
   /components
     SearchBar.tsx
     PaperCard.tsx
     SimilarPapers.tsx
   /lib
     supabase.ts
     openai.ts
   /types
     paper.ts

4. Create .env.local with placeholders:
   NEXT_PUBLIC_SUPABASE_URL=
   NEXT_PUBLIC_SUPABASE_ANON_KEY=
   SUPABASE_SERVICE_KEY=
   OPENAI_API_KEY=

5. Set up Supabase client in /lib/supabase.ts

6. Create TypeScript type in /types/paper.ts:
   interface Paper {
     id: string
     title: string
     authors: string[]
     year: number
     summary_short: string
     summary_long: string
     pdf_url: string
     embedding: number[]
     university?: string
     keywords?: string[]
     language: 'uzbek' | 'russian' | 'english'
     created_at: string
   }

7. Build basic home page UI:
   - Centered search bar (large, prominent)
   - Heading: "Search Uzbek Research Papers with AI"
   - Subheading: "First AI-powered research database for Uzbekistan"
   - Clean, modern design (similar to Google Scholar)

8. Build PaperCard component to display search results:
   - Title (bold, large)
   - Authors (gray, smaller)
   - Year + University
   - Short summary
   - "View Similar" button
   - "Download PDF" button

9. Deploy to Vercel and share URL

DON'T implement search functionality yet - just the UI shell.
Use placeholder data to show how results will look.
```

---

### **DAY 2: Supabase Database + Vector Setup**

```
TASK: Set up Supabase database with vector search capability

1. In Supabase SQL Editor, run this schema:

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE papers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  authors TEXT[] NOT NULL,
  year INTEGER NOT NULL,
  summary_short TEXT NOT NULL,
  summary_long TEXT NOT NULL,
  pdf_url TEXT NOT NULL,
  embedding vector(1536),
  university TEXT,
  keywords TEXT[],
  language TEXT CHECK (language IN ('uzbek', 'russian', 'english')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON papers USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

2. Create Python script: scripts/process_pdf.py

This script should:
- Take PDF file path as input
- Extract text using PyMuPDF (fitz)
- Clean extracted text (remove extra whitespace, page numbers)
- Call OpenAI GPT-4-mini to generate:
  - Clean title (if messy)
  - 2-sentence summary (summary_short)
  - 1-paragraph summary (summary_long)
  - Extract authors, year, keywords
- Generate embedding using OpenAI text-embedding-3-small
- Insert into Supabase papers table

Include error handling and progress logging.

3. Create requirements.txt:
PyMuPDF
openai
python-dotenv
supabase

4. Add script to process entire folder:
scripts/process_folder.py - loops through all PDFs in a directory

5. Test with 2-3 sample PDFs provided by co-founder

Provide clear instructions for co-founder on how to run the script.
```

---

### **DAY 3: Search API + Vector Similarity**

```
TASK: Implement semantic search API using vector similarity

1. Create API route: app/api/search/route.ts

This endpoint should:
- Accept POST request with { query: string, limit?: number }
- Generate embedding for query using OpenAI
- Perform vector similarity search in Supabase:
  SELECT *, 
         1 - (embedding <=> query_embedding) as similarity
  FROM papers
  ORDER BY embedding <=> query_embedding
  LIMIT 10
- Return papers sorted by similarity score

2. Create search utility: lib/search.ts
- generateQueryEmbedding(query: string)
- searchPapers(query: string, limit: number)

3. Update SearchBar component:
- Add loading state
- Call /api/search on submit
- Display results using PaperCard component
- Handle empty results with helpful message

4. Add search result metadata:
- Number of results found
- Search time
- Similarity scores (show as percentage)

5. Implement client-side filters:
- Filter by year range (slider)
- Filter by language (checkbox)
- Filter by university (dropdown)

6. Add URL query params for shareable searches:
- /search?q=irrigation&year=2020-2023&lang=uzbek

Test with various queries in Uzbek, Russian, and English.
Search should work cross-lingually (Uzbek query finds Russian papers if semantically similar).
```

---

### **DAY 4: Similar Papers Feature (Killer Feature)**

```
TASK: Build "Find Similar Papers" feature - the demo showstopper

1. Create API route: app/api/similar/route.ts

This endpoint should:
- Accept POST { paperId: string, limit?: number }
- Fetch the paper's embedding from database
- Find most similar papers using vector search
- Exclude the original paper from results
- Return top 5 similar papers with similarity scores

2. Update PaperCard component:
- Add "View Similar Papers" button
- Opens modal/drawer showing similar papers
- Each similar paper shows similarity percentage

3. Create SimilarPapers component:
- Takes paperId as prop
- Fetches similar papers
- Displays in elegant grid/list
- Shows why papers are similar (matching keywords highlighted)

4. Add "Paper Detail" page: app/paper/[id]/page.tsx
- Full paper information
- Long summary
- Download button
- "Similar Papers" section (automatic)
- Citation format (APA, MLA options)

5. Enhance similarity visualization:
- Show keyword overlap between papers
- Display similarity score as visual bar
- Group similar papers by topic clusters

6. Create demo scenario:
- Upload a well-known paper
- Search for it
- Click "View Similar"
- Show 5 highly relevant results
- This demonstrates the AI working!

Make this feature visually impressive - it's what VCs will remember.
```

---

### **DAY 5: Polish + Admin Dashboard**

```
TASK: Build admin dashboard + polish the demo

1. Create admin page: app/dashboard/page.tsx

Features:
- Upload new PDF (drag & drop)
- Shows processing status
- List all papers in database
- Edit paper metadata (title, authors, etc)
- Delete paper
- Bulk upload (process folder of PDFs)
- Simple stats:
  - Total papers
  - Papers by language
  - Papers by year
  - Most searched topics (if you track searches)

2. Add search analytics:
- Track searches in Supabase:
  CREATE TABLE searches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query TEXT NOT NULL,
    results_count INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
  )
- Show popular searches on dashboard

3. UI/UX polish:
- Add loading skeletons for search results
- Smooth animations (framer-motion)
- Error boundaries for graceful failures
- Toast notifications for actions
- Responsive design (mobile-friendly)

4. Performance optimization:
- Cache search results (React Query or SWR)
- Lazy load search results
- Optimize images
- Add meta tags for SEO

5. Create 3 bookmarkable demo searches:
- /demo/search1 (pre-filled agriculture query)
- /demo/search2 (pre-filled education query)
- /demo/search3 (pre-filled technology query)
- These should show best results immediately

6. Add empty states:
- No search results found
- No similar papers
- Database empty (admin view)

Make everything feel fast and professional - even with small dataset.
```

---

### **DAY 6: Demo Scenarios + Edge Cases**

```
TASK: Prepare 3 perfect demo scenarios and handle edge cases

1. Create demo page: app/demo/page.tsx

Include 3 pre-set demo buttons:
- "Demo 1: Search Agriculture Papers" → auto-fills query, shows results
- "Demo 2: Find Similar Papers" → shows a paper + its similar matches
- "Demo 3: Cross-language Search" → Uzbek query finding Russian papers

2. Handle edge cases:
- PDF with no text (scanned image) → show error message
- Very long paper → truncate summary intelligently  
- Paper with no authors → display as "Unknown"
- Duplicate papers → detect and mark as duplicate
- Broken PDF URL → show "Download unavailable"

3. Add helpful features:
- Highlight search terms in results
- Show "You might also search for..." suggestions
- Add recent searches (localStorage)
- Keyboard shortcuts (/ to focus search)

4. Create comparison view:
- Select 2-3 papers
- Show side-by-side comparison
- Highlight similarities and differences
- Generate AI summary of how papers relate

5. Improve error messages:
- "No results found. Try broader terms like..."
- "Processing PDF... this may take 30 seconds"
- "Upload failed. Ensure PDF is under 50MB"

6. Add quick stats on homepage:
- "X papers indexed"
- "Y universities covered"
- "Z research topics"

Test all 3 demo scenarios 10+ times. They must work flawlessly.
```

---

### **DAY 7: Final Testing + Deployment**

```
TASK: Final polish, testing, and deployment prep

1. Critical bug fixes only:
- Test every button
- Test on Chrome, Safari, Firefox
- Test on mobile devices
- Fix any breaking issues

2. Performance check:
- Run Lighthouse audit
- Ensure search responds < 2 seconds
- Optimize bundle size
- Check Vercel deployment logs

3. Add minimal analytics:
- Vercel Analytics (built-in)
- Track page views
- Track search queries (anonymized)

4. Create backup plan:
- If live demo fails, have recorded video ready
- Have localhost version running as backup
- Export database snapshot

5. Documentation:
- Create README with:
  - How to add papers
  - How search works
  - Future roadmap
- Screenshot collection for pitch deck

6. Final deployment checklist:
- [ ] Environment variables set in Vercel
- [ ] Database has 20+ papers
- [ ] All demo scenarios work
- [ ] PDF downloads work
- [ ] Mobile responsive
- [ ] Error handling works
- [ ] Similar papers feature works
- [ ] Admin dashboard locked (password protect)

7. Prepare handoff to co-founder:
- Admin credentials
- How to add papers
- How to fix common issues
- Demo walkthrough script

DO NOT add new features. Only fix critical bugs.

Deploy final version and lock it - no more changes before demo!
```

---

## **🚀 Quick Start Command for Junior Devs**

Day 1 Morning:
```bash
npx create-next-app@latest ilmai --typescript --tailwind --app
cd ilmai
npm install @supabase/supabase-js openai react-icons
# Then paste Day 1 Cursor prompt
```

**Give these to your team NOW. Start Day 1 today.**