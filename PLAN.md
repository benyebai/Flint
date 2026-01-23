# Flint - System Design & Implementation Plan

> **PR Semantic Analysis Tool** - Help code reviewers understand the meaning and importance of code changes.

## Table of Contents

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Component Design](#3-component-design)
4. [Data Flow](#4-data-flow)
5. [Database Schema](#5-database-schema)
6. [API Specification](#6-api-specification)
7. [Key Algorithms](#7-key-algorithms)
8. [Chrome Extension UI](#8-chrome-extension-ui)
9. [Implementation Plan](#9-implementation-plan)
10. [Tech Stack](#10-tech-stack)

---

## 1. Overview

### What is Flint?

Flint is a Chrome extension + backend service that analyzes GitHub Pull Requests and provides:

- **Semantic explanations** for each code change (what it does, why it matters)
- **Importance scoring** (🔴 High / 🟡 Medium / 🟢 Low) with reasoning
- **Review hints** (what reviewers should pay attention to)

### Key Features

| Feature | Description |
|---------|-------------|
| **Deep Context Gathering** | Clone repo in Vercel Sandbox, run ripgrep to find callers, tests, types |
| **Importance Scoring** | Heuristic-based scoring considering centrality, security, API surface |
| **LLM Semantic Analysis** | Claude-powered explanations of what each change means |
| **GitHub Integration** | Inline overlays injected directly into GitHub's PR diff view |

### Target Users

- **Code Reviewers** - Understand PRs faster, focus on what matters
- **Team Leads** - Quick overview of PR risk and scope
- **New Team Members** - Learn codebase through contextual explanations

---

## 2. System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    FLINT SYSTEM                                          │
│                                                                                          │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐                   │
│  │  Chrome Ext     │────▶│  Next.js API    │────▶│  Vercel Sandbox │                   │
│  │  (Frontend)     │◀────│  (Backend)      │◀────│  (Compute)      │                   │
│  └─────────────────┘     └─────────────────┘     └─────────────────┘                   │
│          │                       │                       │                              │
│          │                       ▼                       │                              │
│          │               ┌─────────────────┐             │                              │
│          │               │  PostgreSQL     │             │                              │
│          │               │  (Cache/State)  │             │                              │
│          │               └─────────────────┘             │                              │
│          │                       │                       │                              │
│          │                       ▼                       │                              │
│          └──────────────▶│  Claude API     │◀────────────┘                              │
│                          │  (LLM)          │                                            │
│                          └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### Why Vercel Sandbox?

We need deep context to understand code changes:

| Approach | Capability | Limitation |
|----------|------------|------------|
| **GitHub API only** | Fetch files on demand | Rate limits, slow for many files, no regex search |
| **Vercel Sandbox** | Clone repo, run ripgrep, full filesystem | Ephemeral, costs per use |

Vercel Sandbox gives us:
- Full repo clone (shallow, fast)
- Run `ripgrep` for instant code search
- Parse AST, resolve imports
- No rate limits within the sandbox

---

## 3. Component Design

### 3.1 Chrome Extension

```
extension/
├── manifest.json              # Manifest V3
├── src/
│   ├── background.ts          # Service worker (auth, API calls)
│   ├── content.tsx            # Content script (GitHub injection)
│   ├── popup.tsx              # Extension popup (settings, login)
│   ├── components/
│   │   ├── SemanticPanel.tsx  # Main overlay panel
│   │   ├── ImportanceBadge.tsx # 🔴🟡🟢 badge with score
│   │   └── LoadingState.tsx   # Loading skeleton
│   ├── lib/
│   │   ├── github.ts          # GitHub page detection
│   │   ├── api.ts             # Backend API client
│   │   └── storage.ts         # Chrome storage helpers
│   └── styles/
│       └── overlay.css        # Styles for injected UI
```

**Responsibilities:**
- Detect GitHub PR pages
- Find diff hunks in DOM
- Call backend API for analysis
- Inject overlay panels on each hunk

### 3.2 Backend (Next.js)

```
apps/web/
├── app/
│   ├── api/
│   │   ├── auth/
│   │   │   ├── github/route.ts    # OAuth callback
│   │   │   └── session/route.ts   # Get session
│   │   ├── analyze/
│   │   │   ├── route.ts           # POST - start analysis
│   │   │   └── [id]/route.ts      # GET - get results
│   │   └── repos/
│   │       └── [owner]/[repo]/route.ts
│   └── page.tsx                   # Landing page
├── lib/
│   ├── sandbox/
│   │   ├── manager.ts             # Create/manage sandboxes
│   │   ├── context.ts             # Context gathering logic
│   │   └── commands.ts            # Sandbox command helpers
│   ├── analysis/
│   │   ├── diff.ts                # Diff parsing
│   │   ├── importance.ts          # Importance scoring
│   │   └── semantic.ts            # LLM analysis
│   ├── db/
│   │   ├── schema.ts              # Drizzle schema
│   │   └── queries.ts             # Database queries
│   └── auth/
│       └── github.ts              # GitHub OAuth helpers
```

### 3.3 Analysis Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ANALYSIS PIPELINE                                     │
│                                                                              │
│  Request → SandboxManager → DiffParser → ContextGatherer → ImportanceScorer │
│                                                      ↓                       │
│                                              SemanticAnalyzer → Response     │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Stage | Input | Output | Where |
|-------|-------|--------|-------|
| **SandboxManager** | PR info, token | Sandbox instance | Vercel Sandbox |
| **DiffParser** | `git diff` output | Structured hunks | Sandbox |
| **ContextGatherer** | Hunks | Context per hunk | Sandbox |
| **ImportanceScorer** | Hunks + context | Scores + tags | Backend |
| **SemanticAnalyzer** | Hunks + context + scores | Explanations | Claude API |

---

## 4. Data Flow

### Full Analysis Flow

```
User clicks "Analyze" on GitHub PR
              │
              ▼
┌──────────────────────────────────────┐
│  Extension: Extract PR info          │
│  { owner, repo, prNumber, branch }   │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Backend: POST /api/analyze          │
│  Return analysisId immediately       │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Sandbox: Clone repo (shallow)       │
│                                      │
│  Sandbox.create({                    │
│    source: { type: 'git', ... },     │
│    depth: 1,                         │
│    revision: prBranch                │
│  })                                  │
│                                      │
│  Time: ~10-30 seconds                │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Sandbox: Get diff                   │
│  git diff origin/main...HEAD         │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Sandbox: Gather context (parallel)  │
│                                      │
│  For each changed file:              │
│  ├─ rg "import.*{file}" → callers    │
│  ├─ Read containing function         │
│  ├─ Find related tests               │
│  └─ Load AGENTS.md guidelines        │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Sandbox: Stop (cleanup)             │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Backend: Score importance           │
│  (heuristic calculation)             │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Backend: LLM semantic analysis      │
│  (Claude API, batched)               │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Backend: Cache results in DB        │
│  Return to extension                 │
└──────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Extension: Inject overlays          │
│  on each diff hunk in GitHub UI      │
└──────────────────────────────────────┘
```

### Context Gathering Detail

```
                              ┌─────────────────┐
                              │  Changed File   │
                              │  src/auth.ts    │
                              └────────┬────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
           ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│  Find Callers       │   │  Find Tests         │   │  Read Guidelines    │
│                     │   │                     │   │                     │
│  rg "import.*auth"  │   │  find . -name       │   │  cat AGENTS.md      │
│  --json             │   │  "auth.test.*"      │   │                     │
│                     │   │                     │   │                     │
│  Result:            │   │  Result:            │   │  Result:            │
│  • src/login.ts     │   │  • auth.test.ts     │   │  "Review security   │
│  • src/session.ts   │   │                     │   │   changes carefully"│
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
           │                           │                           │
           └───────────────────────────┼───────────────────────────┘
                                       │
                                       ▼
                        ┌─────────────────────────┐
                        │  Combined Context       │
                        │                         │
                        │  {                      │
                        │    callers: [3 files],  │
                        │    tests: [2 files],    │
                        │    guidelines: "...",   │
                        │    imports: [...],      │
                        │    containingFn: "..."  │
                        │  }                      │
                        └─────────────────────────┘
```

---

## 5. Database Schema

```sql
-- Users (GitHub OAuth)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  github_id TEXT UNIQUE NOT NULL,
  github_username TEXT NOT NULL,
  github_token TEXT NOT NULL,  -- encrypted
  email TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Repositories (for snapshot caching)
CREATE TABLE repositories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner TEXT NOT NULL,
  repo TEXT NOT NULL,
  
  -- Vercel Sandbox snapshot for fast restarts
  snapshot_id TEXT,
  snapshot_created_at TIMESTAMP,
  snapshot_expires_at TIMESTAMP,  -- 7 days
  
  -- Cached metadata
  default_branch TEXT DEFAULT 'main',
  guidelines TEXT,  -- AGENTS.md content
  
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(owner, repo)
);

-- PR Analyses (cached results)
CREATE TABLE analyses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- PR identification
  owner TEXT NOT NULL,
  repo TEXT NOT NULL,
  pr_number INT NOT NULL,
  commit_sha TEXT NOT NULL,
  
  -- Status
  status TEXT NOT NULL DEFAULT 'pending',
  -- 'pending' | 'running' | 'completed' | 'failed'
  error TEXT,
  
  -- Results
  result JSONB,
  /*
    {
      hunks: [{
        file, startLine, endLine,
        semantic: { summary, intent, behaviorChange, implications, reviewHints },
        importance: { score, level, factors, tags },
        context: { callers, tests, containingFunction }
      }],
      metadata: { totalFiles, totalHunks, analysisTime }
    }
  */
  
  user_id UUID REFERENCES users(id),
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(owner, repo, pr_number, commit_sha)
);

-- Indexes
CREATE INDEX idx_analyses_pr ON analyses(owner, repo, pr_number);
CREATE INDEX idx_analyses_status ON analyses(status) WHERE status = 'running';
```

---

## 6. API Specification

### Authentication

```
POST /api/auth/github
  Request:  { code: string }
  Response: { user: { id, githubUsername }, token: string }

GET /api/auth/session
  Headers:  Authorization: Bearer {token}
  Response: { user: { id, githubUsername } }

POST /api/auth/logout
  Headers:  Authorization: Bearer {token}
  Response: { success: true }
```

### Analysis

```
POST /api/analyze
  Headers:  Authorization: Bearer {token}
  Request:  { owner, repo, prNumber, branch }
  Response: { analysisId, status: "pending" }

GET /api/analyze/{id}
  Headers:  Authorization: Bearer {token}
  Response:
    If completed:
      {
        id, status: "completed",
        result: {
          hunks: [{
            file, startLine, endLine,
            semantic: { summary, intent, behaviorChange, implications, reviewHints },
            importance: { score, level, factors, tags }
          }],
          metadata: { totalFiles, totalHunks, analysisTime }
        }
      }
    If running:
      { id, status: "running", progress: { phase, current, total } }
```

---

## 7. Key Algorithms

### 7.1 Importance Scoring

```
FACTORS (0-100 total):

1. CENTRALITY (0-30 points)
   - callers > 10  → +25 "high_centrality"
   - callers > 5   → +15 "medium_centrality"
   - callers > 0   → +5  "some_callers"

2. SECURITY (0-30 points)
   - file matches /auth|security|crypto|password|token/
     → +30 "security_sensitive"

3. API SURFACE (0-25 points)
   - changes exported function  → +25 "public_api"
   - changes function signature → +20 "breaking_change_risk"

4. TEST COVERAGE (0-15 points)
   - no related tests → +15 "untested"

5. CODE SMELL (0-15 points)
   - contains TODO/FIXME → +10 "incomplete"
   - large change (>50 lines) → +10 "large_change"

SCORE = min(100, sum(matched_factors))
LEVEL = score >= 60 ? "high" : score >= 30 ? "medium" : "low"
```

### 7.2 LLM Prompt Structure

**System Prompt:**
```
You are Flint, a code review assistant. Analyze code changes and
explain their semantics for code reviewers.

Repository Guidelines:
{guidelines from AGENTS.md}

Your analysis must include:
1. Summary: 1-2 sentences explaining the change
2. Intent: What the developer was trying to achieve
3. Behavior Change: What's different after this change
4. Implications: Potential side effects (array)
5. Review Hints: What reviewers should check (array)

Respond in JSON format. Be concise and technical.
```

**User Prompt (per hunk):**
```
## File: {filePath}

## Diff:
\`\`\`diff
{unified diff}
\`\`\`

## Containing Function:
\`\`\`{language}
{function code}
\`\`\`

## Files that import this ({count}):
{caller list}

## Related Tests:
{test files}

## Importance Score: {score}/100 ({level})
Factors: {tags}

Analyze this change.
```

---

## 8. Chrome Extension UI

### Overlay Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│  🔴 75/100  [security] [public_api]                                     │
│                                                                         │
│  Adds JWT token validation with expiry checking.                        │
│                                                                         │
│  ▼ Details                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Intent: Add token expiration validation to prevent stale      │   │
│  │  sessions from accessing protected resources.                   │   │
│  │                                                                 │   │
│  │  Behavior Change: Requests with expired tokens will now        │   │
│  │  return 401 instead of being processed.                         │   │
│  │                                                                 │   │
│  │  Review Checklist:                                              │   │
│  │  • Verify token expiry time is configurable                     │   │
│  │  • Check error message doesn't leak sensitive info              │   │
│  │  • Ensure refresh token flow is updated                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ▼ Why 75/100?                                                          │
│  • security_sensitive: +30 (auth-related file)                          │
│  • public_api: +25 (changes exported function)                          │
│  • high_centrality: +20 (8 files import this)                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Color Coding

| Level | Color | Score Range |
|-------|-------|-------------|
| 🔴 High | Red (#dc2626) | 60-100 |
| 🟡 Medium | Yellow (#f59e0b) | 30-59 |
| 🟢 Low | Green (#22c55e) | 0-29 |

---

## 9. Implementation Plan

### Week 1: Backend + Sandbox Integration

| Day | Task | Details |
|-----|------|---------|
| 1 | Project setup | Next.js app, Vercel project, @vercel/sandbox |
| 2 | GitHub OAuth | NextAuth.js, token storage |
| 3 | Sandbox clone | Clone repo, checkout PR branch |
| 4 | Diff parser | Parse git diff into structured hunks |
| 5 | Context gatherer | ripgrep commands, file reading |
| 6 | File parsing | Extract functions, imports, exports |
| 7 | Integration test | End-to-end: PR URL → context |

**Deliverables:**
- [ ] Next.js app deployed to Vercel
- [ ] GitHub OAuth working
- [ ] Sandbox clones repo and gathers context
- [ ] Diff parsed into structured format

### Week 2: Analysis Pipeline + LLM

| Day | Task | Details |
|-----|------|---------|
| 1 | Importance scorer | Implement scoring algorithm |
| 2 | LLM prompt design | System + user prompts |
| 3 | Claude integration | API calls, structured output |
| 4 | Batch processing | Analyze multiple hunks efficiently |
| 5 | Caching layer | PostgreSQL + Redis caching |
| 6 | API endpoints | /api/analyze complete |
| 7 | Error handling | Timeouts, retries |

**Deliverables:**
- [ ] Importance scoring working
- [ ] Claude API integration
- [ ] Results cached in database
- [ ] API returns analysis results

### Week 3: Chrome Extension

| Day | Task | Details |
|-----|------|---------|
| 1 | Extension scaffold | Manifest V3, Plasmo/WXT |
| 2 | GitHub detection | Detect PR pages, extract info |
| 3 | Diff hunk detection | Find hunks in DOM |
| 4 | UI injection | Inject overlay containers |
| 5 | Panel components | SemanticPanel, ImportanceBadge |
| 6 | Auth flow | GitHub OAuth in extension |
| 7 | Polish + testing | Error states, loading |

**Deliverables:**
- [ ] Extension installed and working
- [ ] Overlays injected on GitHub PR pages
- [ ] Full flow working end-to-end

---

## 10. Tech Stack

| Component | Technology | Why |
|-----------|------------|-----|
| **Backend** | Next.js 14+ (App Router) | API routes, edge functions, Vercel integration |
| **Sandbox** | @vercel/sandbox | Clone repos, run commands, managed VMs |
| **Database** | PostgreSQL (Neon) | Relational data, JSONB for results |
| **Cache** | Vercel KV (Redis) | Sessions, rate limiting |
| **Auth** | NextAuth.js | GitHub OAuth |
| **LLM** | Anthropic Claude | Best code understanding |
| **Extension** | Plasmo or WXT | Modern extension framework |
| **Styling** | Tailwind CSS | Fast styling |
| **Hosting** | Vercel | Seamless deployment |

### Cost Estimation

| Item | Estimate |
|------|----------|
| Vercel Sandbox | Free tier, then usage-based |
| Claude API | ~$0.01-0.05 per PR |
| Vercel Hosting | Free → $20/mo Pro |
| Database (Neon) | Free tier |

---

## Appendix: Key Code Snippets

### Sandbox Creation

```typescript
import { Sandbox } from '@vercel/sandbox'
import ms from 'ms'

async function createRepoSandbox(input: {
  owner: string
  repo: string
  branch: string
  token: string
}) {
  return Sandbox.create({
    source: {
      type: 'git',
      url: `https://github.com/${input.owner}/${input.repo}.git`,
      username: 'x-access-token',
      password: input.token,
      depth: 1,
      revision: input.branch,
    },
    runtime: 'node24',
    timeout: ms('5m'),
  })
}
```

### Context Gathering

```typescript
async function gatherContext(sandbox: Sandbox, file: string) {
  const [callers, tests, content] = await Promise.all([
    sandbox.runCommand('rg', ['--json', `import.*from.*${file}`, '.']),
    sandbox.runCommand('find', ['.', '-name', `${basename(file)}.test.*`]),
    sandbox.readFileToBuffer({ path: file }),
  ])
  
  return {
    callers: parseRipgrepJson(await callers.stdout()),
    tests: (await tests.stdout()).split('\n').filter(Boolean),
    content: content?.toString() ?? '',
  }
}
```

### Importance Scoring

```typescript
function scoreImportance(hunk: DiffHunk, context: Context): ImportanceScore {
  const factors = []
  
  if (context.callers.length > 10) {
    factors.push({ name: 'high_centrality', weight: 25, reason: `${context.callers.length} files depend on this` })
  }
  
  if (/auth|security|crypto/i.test(hunk.file)) {
    factors.push({ name: 'security', weight: 30, reason: 'Security-sensitive file' })
  }
  
  if (context.tests.length === 0) {
    factors.push({ name: 'untested', weight: 15, reason: 'No tests found' })
  }
  
  const score = Math.min(100, factors.reduce((sum, f) => sum + f.weight, 0))
  const level = score >= 60 ? 'high' : score >= 30 ? 'medium' : 'low'
  
  return { score, level, factors, tags: factors.map(f => f.name) }
}
```

---

*Last updated: January 2026*
