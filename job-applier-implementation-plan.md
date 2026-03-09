# Auto-Job Applier - Implementation Plan

**Project:** Auto-Job Applier  
**Version:** 1.0  
**Created:** 2026-03-09  
**Scope:** Job Search → Save to Dashboard → One-Click Apply URL → AI Resume Tailoring  
**Tech Stack:** Next.js 14+, Supabase, Tailwind CSS, TypeScript

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Environment Setup](#2-environment-setup)
3. [Database Schema & Migrations](#3-database-schema--migrations)
4. [File Structure](#4-file-structure)
5. [API Routes](#5-api-routes)
6. [Components](#6-components)
7. [Pages](#7-pages)
8. [Services & Utilities](#8-services--utilities)
9. [State Management](#9-state-management)
10. [Authentication](#10-authentication)
11. [Job Discovery & Scraper](#11-job-discovery--scraper)
12. [AI Resume Tailoring](#12-ai-resume-tailoring)
13. [Environment Variables Reference](#13-environment-variables-reference)
14. [Error Handling Strategy](#14-error-handling-strategy)
15. [Testing Approach](#15-testing-approach)
16. [Phase-Based Implementation Guide](#16-phase-based-implementation-guide)

---

## 1. Project Overview

### Core Features (Simplified Scope)

1. **Job Search** - Discover jobs from multiple sources (LinkedIn, Wellfound, Otta, GitHub Jobs)
2. **Save to Dashboard** - Save interesting jobs to personal dashboard
3. **One-Click Apply URL** - Quick access to apply directly on company website
4. **AI Resume Tailoring** - Tailor resume for specific job postings using AI

### What We're NOT Building (Deferred)

- Browser automation for auto-apply
- Email follow-up automation
- Full application tracking with status updates
- Comprehensive analytics dashboard

---

## 2. Environment Setup

### Prerequisites

```bash
# Node.js 20+ required
node --version  # Should be >= 20.0.0

# npm or pnpm
npm --version
pnpm --version  # Recommended

# Supabase CLI (for local development)
npm install -g supabase
supabase --version

# GitHub CLI (for repo management)
gh auth status
```

### Initial Setup Commands

```bash
# 1. Create Next.js project
cd /data/workspace-maker
npx create-next-app@latest job-applier \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --use-npm

# 2. Install core dependencies
cd job-applier
npm install @supabase/supabase-js @supabase/ssr \
  @tanstack/react-query zustand zod \
  lucide-react clsx tailwind-merge \
  date-fns react-hook-form @hookform/resolvers

# 3. Install shadcn/ui (optional but recommended)
npx shadcn-ui@latest init

# 4. Install additional UI dependencies
npm install framer-motion @radix-ui/react-dialog \
  @radix-ui/react-dropdown-menu @radix-ui/react-select \
  @radix-ui/react-tabs @radix-ui/react-toast \
  @radix-ui/react-slot

# 5. Install file handling
npm install react-dropzone pdf-parse docx
```

### Directory Context

All file paths in this document are relative to the project root unless otherwise specified.

---

## 3. Database Schema & Migrations

### Supabase Setup

1. **Create Supabase Project**
   - Go to https://supabase.com
   - Create new project: `job-applier`
   - Note down: `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SERVICE_ROLE_KEY`

2. **Run Migrations**
   Using SQL Editor in Supabase dashboard, execute the following:

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ============================================
-- USERS TABLE (extends Supabase auth.users)
-- ============================================
CREATE TABLE public.profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  target_roles TEXT[] DEFAULT ARRAY['MLOps Engineer', 'AI Platform Engineer', 'SRE', 'DevOps', 'Solutions Engineer'],
  target_location TEXT DEFAULT 'Minneapolis',
  remote_only BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- JOBS TABLE
-- ============================================
CREATE TABLE public.jobs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  source TEXT NOT NULL,
  source_id TEXT NOT NULL,
  title TEXT NOT NULL,
  company TEXT NOT NULL,
  location TEXT,
  remote BOOLEAN DEFAULT false,
  salary_min INTEGER,
  salary_max INTEGER,
  salary_currency TEXT DEFAULT 'USD',
  description TEXT,
  requirements TEXT[],
  nice_to_have TEXT[],
  url TEXT NOT NULL,
  posted_at TIMESTAMPTZ,
  discovered_at TIMESTAMPTZ DEFAULT NOW(),
  status TEXT DEFAULT 'new' CHECK (status IN ('new', 'saved', 'applied', 'archived')),
  match_score INTEGER,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, source, source_id)
);

-- ============================================
-- APPLICATIONS TABLE
-- ============================================
CREATE TABLE public.applications (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  job_id UUID REFERENCES public.jobs(id) ON DELETE SET NULL,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'applied', 'viewed', 'responded', 'interview', 'rejected', 'accepted')),
  applied_at TIMESTAMPTZ,
  application_url TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- RESUMES TABLE
-- ============================================
CREATE TABLE public.resumes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  file_url TEXT,
  content TEXT,
  ats_score INTEGER,
  ats_feedback JSONB,
  keywords_matched TEXT[],
  keywords_missing TEXT[],
  is_default BOOLEAN DEFAULT false,
  version INTEGER DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- COVER LETTERS TABLE
-- ============================================
CREATE TABLE public.cover_letters (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  job_id UUID REFERENCES public.jobs(id) ON DELETE SET NULL,
  application_id UUID REFERENCES public.applications(id) ON DELETE SET NULL,
  content TEXT NOT NULL,
  file_url TEXT,
  is_template BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- TAILORED RESUMES TABLE
-- ============================================
CREATE TABLE public.tailored_resumes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  job_id UUID REFERENCES public.jobs(id) ON DELETE CASCADE,
  original_resume_id UUID REFERENCES public.resumes(id) ON DELETE SET NULL,
  content TEXT NOT NULL,
  matched_keywords TEXT[],
  missing_keywords TEXT[],
  improvements_suggested TEXT[],
  ats_score_before INTEGER,
  ats_score_after INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- ANALYTICS EVENTS TABLE
-- ============================================
CREATE TABLE public.analytics_events (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  event_type TEXT NOT NULL,
  event_data JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- INDEXES
-- ============================================
CREATE INDEX idx_jobs_user_status ON public.jobs(user_id, status);
CREATE INDEX idx_jobs_discovered ON public.jobs(discovered_at DESC);
CREATE INDEX idx_jobs_source ON public.jobs(source);
CREATE INDEX idx_jobs_match_score ON public.jobs(match_score DESC);
CREATE INDEX idx_applications_user_status ON public.applications(user_id, status);
CREATE INDEX idx_applications_job ON public.applications(job_id);
CREATE INDEX idx_resumes_user ON public.resumes(user_id);
CREATE INDEX idx_analytics_user_type ON public.analytics_events(user_id, event_type);
CREATE INDEX idx_analytics_created ON public.analytics_events(created_at DESC);

-- ============================================
-- ROW LEVEL SECURITY (RLS)
-- ============================================

-- Enable RLS
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.jobs ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.applications ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.resumes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.cover_letters ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.tailored_resumes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.analytics_events ENABLE ROW LEVEL SECURITY;

-- Profiles: Users can only access their own profile
CREATE POLICY "Users can select own profile" ON public.profiles
  FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users can update own profile" ON public.profiles
  FOR UPDATE USING (auth.uid() = id);
CREATE POLICY "Users can insert own profile" ON public.profiles
  FOR INSERT WITH CHECK (auth.uid() = id);

-- Jobs: Users can only access their own jobs
CREATE POLICY "Users can select own jobs" ON public.jobs
  FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own jobs" ON public.jobs
  FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can update own jobs" ON public.jobs
  FOR UPDATE USING (auth.uid() = user_id);
CREATE POLICY "Users can delete own jobs" ON public.jobs
  FOR DELETE USING (auth.uid() = user_id);

-- Applications: Users can only access their own applications
CREATE POLICY "Users can select own applications" ON public.applications
  FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own applications" ON public.applications
  FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can update own applications" ON public.applications
  FOR UPDATE USING (auth.uid() = user_id);
CREATE POLICY "Users can delete own applications" ON public.applications
  FOR DELETE USING (auth.uid() = user_id);

-- Resumes: Users can only access their own resumes
CREATE POLICY "Users can select own resumes" ON public.resumes
  FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own resumes" ON public.resumes
  FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can update own resumes" ON public.resumes
  FOR UPDATE USING (auth.uid() = user_id);
CREATE POLICY "Users can delete own resumes" ON public.resumes
  FOR DELETE USING (auth.uid() = user_id);

-- Cover Letters: Users can only access their own cover letters
CREATE POLICY "Users can select own cover letters" ON public.cover_letters
  FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own cover letters" ON public.cover_letters
  FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can update own cover letters" ON public.cover_letters
  FOR UPDATE USING (auth.uid() = user_id);
CREATE POLICY "Users can delete own cover letters" ON public.cover_letters
  FOR DELETE USING (auth.uid() = user_id);

-- Tailored Resumes: Users can only access their own
CREATE POLICY "Users can select own tailored resumes" ON public.tailored_resumes
  FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own tailored resumes" ON public.tailored_resumes
  FOR INSERT WITH CHECK (auth.uid() = user_id);

-- Analytics: Users can only access their own events
CREATE POLICY "Users can select own analytics" ON public.analytics_events
  FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own analytics" ON public.analytics_events
  FOR INSERT WITH CHECK (auth.uid() = user_id);

-- ============================================
-- TRIGGER: Auto-create profile on signup
-- ============================================
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name, avatar_url)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', split_part(NEW.email, '@', 1)),
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- ============================================
-- STORAGE BUCKET: Resumes
-- ============================================
INSERT INTO storage.buckets (id, name, public)
VALUES ('resumes', 'resumes', true);

CREATE POLICY "Users can upload own resumes"
  ON storage.objects FOR INSERT
  WITH CHECK (bucket_id = 'resumes' AND auth.uid()::text = (storage.foldername(name))[1]);

CREATE POLICY "Users can view own resumes"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'resumes' AND auth.uid()::text = (storage.foldername(name))[1]);

CREATE POLICY "Users can delete own resumes"
  ON storage.objects FOR DELETE
  USING (bucket_id = 'resumes' AND auth.uid()::text = (storage.foldername(name))[1]);
```

### Database Type Definitions

Create `src/types/database.ts`:

```typescript
export type Json =
  | string
  | number
  | boolean
  | null
  | { [key: string]: Json | undefined }
  | Json[]

export interface Profile {
  id: string
  email: string
  full_name: string | null
  avatar_url: string | null
  target_roles: string[]
  target_location: string
  remote_only: boolean
  created_at: string
  updated_at: string
}

export interface Job {
  id: string
  user_id: string
  source: 'linkedin' | 'wellfound' | 'otta' | 'github' | 'manual'
  source_id: string
  title: string
  company: string
  location: string | null
  remote: boolean
  salary_min: number | null
  salary_max: number | null
  salary_currency: string
  description: string | null
  requirements: string[] | null
  nice_to_have: string[] | null
  url: string
  posted_at: string | null
  discovered_at: string
  status: 'new' | 'saved' | 'applied' | 'archived'
  match_score: number | null
  metadata: Json
  created_at: string
  updated_at: string
}

export interface Application {
  id: string
  user_id: string
  job_id: string | null
  status: 'pending' | 'applied' | 'viewed' | 'responded' | 'interview' | 'rejected' | 'accepted'
  applied_at: string | null
  application_url: string | null
  notes: string | null
  created_at: string
  updated_at: string
}

export interface Resume {
  id: string
  user_id: string
  name: string
  file_url: string | null
  content: string | null
  ats_score: number | null
  ats_feedback: Json | null
  keywords_matched: string[] | null
  keywords_missing: string[] | null
  is_default: boolean
  version: number
  created_at: string
  updated_at: string
}

export interface CoverLetter {
  id: string
  user_id: string
  job_id: string | null
  application_id: string | null
  content: string
  file_url: string | null
  is_template: boolean
  created_at: string
  updated_at: string
}

export interface TailoredResume {
  id: string
  user_id: string
  job_id: string
  original_resume_id: string | null
  content: string
  matched_keywords: string[]
  missing_keywords: string[]
  improvements_suggested: string[]
  ats_score_before: number | null
  ats_score_after: number | null
  created_at: string
}

export interface AnalyticsEvent {
  id: string
  user_id: string
  event_type: string
  event_data: Json
  created_at: string
}
```

---

## 4. File Structure

```
job-applier/
├── .env.local.example
├── .eslintrc.json
├── .gitignore
├── next.config.js
├── package.json
├── postcss.config.js
├── README.md
├── tailwind.config.ts
├── tsconfig.json
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── globals.css
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   ├── signup/
│   │   │   │   └── page.tsx
│   │   │   └── layout.tsx
│   │   ├── (dashboard)/
│   │   │   ├── dashboard/
│   │   │   │   └── page.tsx
│   │   │   ├── jobs/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── [id]/
│   │   │   │   │   └── page.tsx
│   │   │   │   └── discover/
│   │   │   │       └── page.tsx
│   │   │   ├── applications/
│   │   │   │   └── page.tsx
│   │   │   ├── resumes/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── upload/
│   │   │   │   │   └── page.tsx
│   │   │   │   └── [id]/
│   │   │   │       └── page.tsx
│   │   │   ├── settings/
│   │   │   │   └── page.tsx
│   │   │   └── layout.tsx
│   │   └── api/
│   │       ├── auth/
│   │       │   ├── [...nextauth]/
│   │       │   │   └── route.ts
│   │       │   ├── callback/
│   │       │   │   └── route.ts
│   │       │   └── profile/
│   │       │       └── route.ts
│   │       ├── jobs/
│   │       │   ├── route.ts
│   │       │   ├── [id]/
│   │       │   │   └── route.ts
│   │       │   ├── discover/
│   │       │   │   └── route.ts
│   │       │   ├── save/
│   │       │   │   └── route.ts
│   │       │   └── stats/
│   │       │       └── route.ts
│   │       ├── applications/
│   │       │   ├── route.ts
│   │       │   └── [id]/
│   │       │       └── route.ts
│   │       ├── resumes/
│   │       │   ├── route.ts
│   │       │   ├── [id]/
│   │       │   │   └── route.ts
│   │       │   ├── upload/
│   │       │   │   └── route.ts
│   │       │   └── tailor/
│   │       │       └── route.ts
│   │       ├── cover-letters/
│   │       │   ├── route.ts
│   │       │   └── generate/
│   │       │       └── route.ts
│   │       └── analytics/
│   │           └── route.ts
│   ├── components/
│   │   ├── ui/
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── card.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── select.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── toast.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── avatar.tsx
│   │   │   └── label.tsx
│   │   ├── auth/
│   │   │   ├── auth-form.tsx
│   │   │   └── oauth-buttons.tsx
│   │   ├── layout/
│   │   │   ├── header.tsx
│   │   │   ├── sidebar.tsx
│   │   │   ├── mobile-nav.tsx
│   │   │   └── footer.tsx
│   │   ├── jobs/
│   │   │   ├── job-card.tsx
│   │   │   ├── job-list.tsx
│   │   │   ├── job-filters.tsx
│   │   │   ├── job-details.tsx
│   │   │   ├── job-match-score.tsx
│   │   │   └── discover-jobs-button.tsx
│   │   ├── applications/
│   │   │   ├── application-card.tsx
│   │   │   ├── application-form.tsx
│   │   │   └── application-status-badge.tsx
│   │   ├── resumes/
│   │   │   ├── resume-upload.tsx
│   │   │   ├── resume-card.tsx
│   │   │   ├── resume-list.tsx
│   │   │   ├── ats-score.tsx
│   │   │   └── tailor-resume-button.tsx
│   │   ├── cover-letters/
│   │   │   ├── cover-letter-editor.tsx
│   │   │   └── generate-cover-letter.tsx
│   │   └── dashboard/
│   │       ├── stats-card.tsx
│   │       ├── recent-applications.tsx
│   │       ├── upcoming-follow-ups.tsx
│   │       └── quick-actions.tsx
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts
│   │   │   ├── server.ts
│   │   │   └── admin.ts
│   │   ├── utils.ts
│   │   ├── hooks/
│   │   │   ├── use-auth.ts
│   │   │   ├── use-jobs.ts
│   │   │   ├── use-applications.ts
│   │   │   ├── use-resumes.ts
│   │   │   └── use-analytics.ts
│   │   ├── validations/
│   │   │   ├── auth.ts
│   │   │   ├── jobs.ts
│   │   │   ├── applications.ts
│   │   │   ├── resumes.ts
│   │   │   └── cover-letters.ts
│   │   └── constants.ts
│   ├── services/
│   │   ├── openai.ts
│   │   ├── scrapers/
│   │   │   ├── linkedin.ts
│   │   │   ├── wellfound.ts
│   │   │   ├── otta.ts
│   │   │   └── github-jobs.ts
│   │   ├── job-parser.ts
│   │   └── resume-parser.ts
│   ├── stores/
│   │   ├── auth-store.ts
│   │   ├── jobs-store.ts
│   │   └── ui-store.ts
│   ├── types/
│   │   ├── database.ts
│   │   ├── jobs.ts
│   │   ├── applications.ts
│   │   └── resumes.ts
│   └── middleware.ts
├── public/
│   └── images/
│       ├── logo.svg
│       └── empty-state.svg
└── supabase/
    └── migrations/
        └── 001_initial_schema.sql
```

---

## 5. API Routes

### 5.1 Authentication API

#### POST /api/auth/signup
**Description:** Register a new user with email/password

**Request Body:**
```typescript
{
  email: string;        // Required, valid email format
  password: string;     // Required, min 8 chars
  fullName?: string;    // Optional
  targetRoles?: string[]; // Default: ['MLOps Engineer', 'AI Platform Engineer', 'SRE', 'DevOps', 'Solutions Engineer']
  targetLocation?: string; // Default: 'Minneapolis'
  remoteOnly?: boolean; // Default: true
}
```

**Success Response (201):**
```typescript
{
  user: { id: string; email: string };
  session: { access_token: string; refresh_token: string };
}
```

**Error Responses:**
- 400: Invalid email format or password too weak
- 409: User already exists

**Validation (Zod):**
```typescript
// src/lib/validations/auth.ts
import { z } from 'zod';

export const signupSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  fullName: z.string().max(100).optional(),
  targetRoles: z.array(z.string()).optional(),
  targetLocation: z.string().optional(),
  remoteOnly: z.boolean().optional()
});

export const signinSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(1, 'Password is required')
});
```

**Implementation:** `src/app/api/auth/signup/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase/server'
import { z } from 'zod'
import { signupSchema } from '@/lib/validations/auth'

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    
    // Validate request body
    const validatedData = signupSchema.parse(body)
    
    const supabase = createClient()
    
    // Sign up user
    const { data, error } = await supabase.auth.signUp({
      email: validatedData.email,
      password: validatedData.password,
      options: {
        data: {
          full_name: validatedData.fullName,
          target_roles: validatedData.targetRoles,
          target_location: validatedData.targetLocation,
          remote_only: validatedData.remoteOnly
        }
      }
    })
    
    if (error) {
      if (error.message.includes('already registered')) {
        return NextResponse.json({ error: 'User already exists' }, { status: 409 })
      }
      throw error
    }
    
    return NextResponse.json({
      user: data.user,
      session: data.session
    }, { status: 201 })
    
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      )
    }
    
    console.error('Signup error:', error)
    return NextResponse.json({ error: 'Failed to create account' }, { status: 500 })
  }
}
```

#### POST /api/auth/signin
**Description:** Sign in with email/password

**Request Body:**
```typescript
{
  email: string;
  password: string;
}
```

**Success Response (200):**
```typescript
{
  user: { id: string; email: string };
  session: { access_token: string; refresh_token: string };
}
```

**Error Responses:**
- 401: Invalid credentials
- 429: Rate limited

#### POST /api/auth/signout
**Description:** Sign out current user

**Headers:** Authorization: Bearer \<token\>

**Success Response (200):**
```typescript
{ success: true }
```

#### GET /api/auth/me
**Description:** Get current authenticated user with profile

**Headers:** Authorization: Bearer \<token\>

**Success Response (200):**
```typescript
{
  user: {
    id: string;
    email: string;
    profile: {
      fullName: string;
      targetRoles: string[];
      targetLocation: string;
      remoteOnly: boolean;
    }
  }
}
```

#### PUT /api/auth/profile
**Description:** Update user profile

**Request Body:**
```typescript
{
  fullName?: string;
  targetRoles?: string[];
  targetLocation?: string;
  remoteOnly?: boolean;
}
```

---

### 5.2 Jobs API

#### GET /api/jobs
**Description:** List jobs with filters

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| status | string | - | Filter by status: new, saved, applied, archived |
| source | string | - | Filter by source: linkedin, wellfound, otta, github |
| search | string | - | Search in title/company |
| remote | boolean | - | Filter remote jobs only |
| minSalary | number | - | Minimum salary |
| maxSalary | number | - | Maximum salary |
| page | number | 1 | Page number |
| limit | number | 20 | Items per page (max 50) |
| sort | string | discovered_at | Sort field |
| order | string | desc | Sort order: asc, desc |

**Success Response (200):**
```typescript
{
  jobs: Job[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  }
}
```

**Implementation:** `src/app/api/jobs/route.ts`
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase/server'
import { z } from 'zod'

const querySchema = z.object({
  status: z.enum(['new', 'saved', 'applied', 'archived']).optional(),
  source: z.enum(['linkedin', 'wellfound', 'otta', 'github', 'manual']).optional(),
  search: z.string().optional(),
  remote: z.coerce.boolean().optional(),
  minSalary: z.coerce.number().optional(),
  maxSalary: z.coerce.number().optional(),
  page: z.coerce.number().min(1).default(1),
  limit: z.coerce.number().min(1).max(50).default(20),
  sort: z.enum(['discovered_at', 'posted_at', 'match_score', 'created_at']).default('discovered_at'),
  order: z.enum(['asc', 'desc']).default('desc')
})

export async function GET(request: NextRequest) {
  try {
    const supabase = createClient()
    
    // Get current user
    const { data: { user }, error: authError } = await supabase.auth.getUser()
    
    if (authError || !user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    
    const { searchParams } = new URL(request.url)
    const params = querySchema.parse({
      status: searchParams.get('status'),
      source: searchParams.get('source'),
      search: searchParams.get('search'),
      remote: searchParams.get('remote'),
      minSalary: searchParams.get('minSalary'),
      maxSalary: searchParams.get('maxSalary'),
      page: searchParams.get('page'),
      limit: searchParams.get('limit'),
      sort: searchParams.get('sort'),
      order: searchParams.get('order')
    })
    
    const { page, limit, sort, order, ...filters } = params
    const offset = (page - 1) * limit
    
    // Build query
    let query = supabase
      .from('jobs')
      .select('*', { count: 'exact' })
      .eq('user_id', user.id)
    
    // Apply filters
    if (filters.status) query = query.eq('status', filters.status)
    if (filters.source) query = query.eq('source', filters.source)
    if (filters.remote !== undefined) query = query.eq('remote', filters.remote)
    if (filters.search) query = query.or(`title.ilike.%${filters.search}%,company.ilike.%${filters.search}%`)
    if (filters.minSalary) query = query.gte('salary_max', filters.minSalary)
    if (filters.maxSalary) query = query.lte('salary_min', filters.maxSalary)
    
    // Apply sorting
    query = query.order(sort, { ascending: order === 'asc' })
    
    // Apply pagination
    query = query.range(offset, offset + limit - 1)
    
    const { data: jobs, error, count } = await query
    
    if (error) {
      console.error('Error fetching jobs:', error)
      return NextResponse.json({ error: 'Failed to fetch jobs' }, { status: 500 })
    }
    
    return NextResponse.json({
      jobs: jobs || [],
      pagination: {
        page,
        limit,
        total: count || 0,
        totalPages: Math.ceil((count || 0) / limit)
      }
    })
    
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Invalid query parameters', details: error.errors },
        { status: 400 }
      )
    }
    
    console.error('Jobs API error:', error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

#### POST /api/jobs
**Description:** Save a new job manually

**Request Body:**
```typescript
{
  title: string;           // Required
  company: string;          // Required
  location?: string;
  remote?: boolean;
  salaryMin?: number;
  salaryMax?: number;
  salaryCurrency?: string;
  description?: string;
  requirements?: string[];
  niceToHave?: string[];
  url: string;              // Required
  source?: string;          // Default: 'manual'
  postedAt?: string;        // ISO date string
}
```

**Success Response (201):**
```typescript
{
  job: Job;
}
```

#### GET /api/jobs/[id]
**Description:** Get job details by ID

**Path Parameters:** id (UUID)

**Success Response (200):**
```typescript
{
  job: Job;
}
```

**Error Responses:**
- 404: Job not found
- 403: Not authorized

#### POST /api/jobs/[id]/save
**Description:** Save a job to user's list

**Path Parameters:** id (UUID)

**Success Response (200):**
```typescript
{
  success: true;
  job: Job;
}
```

#### POST /api/jobs/discover
**Description:** Trigger job discovery from sources

**Request Body:**
```typescript
{
  sources?: string[];        // ['linkedin', 'wellfound', 'otta', 'github']
  targetRoles?: string[];
  location?: string;
  remoteOnly?: boolean;
}
```

**Success Response (200):**
```typescript
{
  success: boolean;
  jobsFound: number;
  jobsSaved: number;
  duplicatesSkipped: number;
  message: string;
}
```

---

### 5.3 Applications API

#### GET /api/applications
**Description:** List user's applications

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| status | string | - | Filter by status |
| page | number | 1 | Page number |
| limit | number | 20 | Items per page |

**Success Response (200):**
```typescript
{
  applications: Application[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  }
}
```

#### POST /api/applications
**Description:** Create a new application

**Request Body:**
```typescript
{
  jobId: string;              // UUID
  resumeId?: string;         // UUID
  coverLetterId?: string;    // UUID
  notes?: string;
}
```

**Success Response (201):**
```typescript
{
  application: Application;
}
```

#### GET /api/applications/[id]
**Description:** Get application details

**Path Parameters:** id (UUID)

**Success Response (200):**
```typescript
{
  application: Application;
  job?: Job;
  resume?: Resume;
}
```

#### PUT /api/applications/[id]
**Description:** Update application

**Request Body:**
```typescript
{
  status?: 'pending' | 'applied' | 'viewed' | 'responded' | 'interview' | 'rejected' | 'accepted';
  notes?: string;
  appliedAt?: string;
}
```

---

### 5.4 Resumes API

#### GET /api/resumes
**Description:** List user's resumes

**Success Response (200):**
```typescript
{
  resumes: Resume[];
}
```

#### POST /api/resumes/upload
**Description:** Upload a new resume

**Content-Type:** multipart/form-data

**Form Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| file | File | Yes | PDF or DOCX file |
| name | string | Yes | Resume display name |
| isDefault | boolean | No | Set as default resume |

**Success Response (201):**
```typescript
{
  resume: Resume;
  uploadUrl: string;
}
```

#### POST /api/resumes/tailor
**Description:** Tailor resume for a specific job using AI

**Request Body:**
```typescript
{
  resumeId: string;     // UUID
  jobId: string;        // UUID
}
```

**Success Response (200):**
```typescript
{
  tailoredResume: {
    id: string;
    content: string;
    matchedKeywords: string[];
    missingKeywords: string[];
    improvementsSuggested: string[];
    atsScoreBefore: number;
    atsScoreAfter: number;
  }
}
```

**Error Responses:**
- 400: Invalid resume or job ID
- 404: Resume or job not found
- 500: AI processing failed

#### GET /api/resumes/[id]
**Description:** Get resume details

**Path Parameters:** id (UUID)

**Success Response (200):**
```typescript
{
  resume: Resume;
  tailoredVersions?: TailoredResume[];
}
```

---

### 5.5 Cover Letters API

#### GET /api/cover-letters
**Description:** List user's cover letters

**Success Response (200):**
```typescript
{
  coverLetters: CoverLetter[];
}
```

#### POST /api/cover-letters/generate
**Description:** Generate AI cover letter for a job

**Request Body:**
```typescript
{
  jobId: string;        // UUID
  resumeId?: string;    // UUID (optional)
  tone?: 'professional' | 'friendly' | 'enthusiastic';
}
```

**Success Response (201):**
```typescript
{
  coverLetter: {
    id: string;
    content: string;
    jobId: string;
  }
}
```

---

### 5.6 Analytics API

#### GET /api/analytics
**Description:** Get user analytics overview

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| period | string | 30d | Period: 7d, 30d, 90d, all |

**Success Response (200):**
```typescript
{
  analytics: {
    totalApplications: number;
    totalJobsSaved: number;
    applicationsThisWeek: number;
    responseRate: number;
    interviewRate: number;
    applicationsBySource: Record<string, number>;
    applicationsOverTime: { date: string; count: number }[];
  }
}
```

---

## 6. Components

### 6.1 UI Components

All components in `src/components/ui/` follow shadcn/ui conventions. Key components:

#### Button (`src/components/ui/button.tsx`)
- Variants: default, destructive, outline, secondary, ghost, link
- Sizes: default, sm, lg, icon
- States: default, hover, active, disabled, loading

#### Input (`src/components/ui/input.tsx`)
- Standard text input
- Types: text, email, password, search
- States: default, focus, error, disabled

#### Card (`src/components/ui/card.tsx`)
- Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter

#### Dialog (`src/components/ui/dialog.tsx`)
- Dialog, DialogTrigger, DialogContent, DialogHeader, DialogTitle, DialogDescription, DialogFooter

#### Badge (`src/components/ui/badge.tsx`)
- Variants: default, secondary, outline, destructive

### 6.2 Auth Components

#### AuthForm (`src/components/auth/auth-form.tsx`)
- Props: mode ('signin' | 'signup'), onSubmit, isLoading
- Internal state: email, password, fullName
- Handles form validation with react-hook-form + zod
- Shows error messages inline

#### OAuthButtons (`src/components/auth/oauth-buttons.tsx`)
- Props: onSuccess, onError
- Providers: GitHub, Google
- Shows loading state during OAuth flow

### 6.3 Layout Components

#### Header (`src/components/layout/header.tsx`)
- Navigation links (desktop)
- User avatar dropdown (profile, settings, sign out)
- Mobile menu toggle
- Sticky positioning

#### Sidebar (`src/components/layout/sidebar.tsx`)
- Navigation items with icons
- Collapsible on mobile
- Active state highlighting
- Items: Dashboard, Jobs, Applications, Resumes, Settings

#### MobileNav (`src/components/layout/mobile-nav.tsx`)
- Bottom navigation for mobile
- Icons + labels
- Active state

### 6.4 Job Components

#### JobCard (`src/components/jobs/job-card.tsx`)
- Props: job: Job, onSave: () => void, onApply: () => void
- Displays: title, company, location, salary, match score, source badge
- Actions: Save button, Apply button (opens URL)
- Hover state with subtle elevation

#### JobList (`src/components/jobs/job-list.tsx`)
- Props: jobs: Job[], isLoading: boolean, onJobClick: (job) => void
- Displays grid/list of JobCard components
- Empty state when no jobs
- Loading state with skeleton cards

#### JobFilters (`src/components/jobs/job-filters.tsx`)
- Props: filters, onFilterChange
- Filter options: status, source, remote, salary range
- Search input
- Clear filters button

#### JobDetails (`src/components/jobs/job-details.tsx`)
- Props: job: Job, isOpen: boolean, onClose: () => void
- Full job details in modal/drawer
- Sections: Header, Description, Requirements, Match Score, Actions

#### JobMatchScore (`src/components/jobs/job-match-score.tsx`)
- Props: score: number, matchedKeywords: string[], missingKeywords: string[]
- Visual score indicator (progress bar)
- Lists matched/missing keywords

#### DiscoverJobsButton (`src/components/jobs/discover-jobs-button.tsx`)
- Props: onClick, isLoading, lastDiscoverd
- Shows loading spinner during discovery
- Displays "Last discovered: X hours ago"

### 6.5 Application Components

#### ApplicationCard (`src/components/applications/application-card.tsx`)
- Props: application: Application with job relation
- Displays: job title, company, status badge, applied date
- Click to view details

#### ApplicationForm (`src/components/applications/application-form.tsx`)
- Props: job: Job, onSubmit: (data) => void, isLoading: boolean
- Resume selector dropdown
- Cover letter toggle + editor
- Submit/Cancel buttons

#### ApplicationStatusBadge (`src/components/applications/application-status-badge.tsx`)
- Props: status: Application['status']
- Color-coded badges:
  - pending: yellow
  - applied: blue
  - viewed: purple
  - responded: green
  - interview: green (bold)
  - rejected: red
  - accepted: green (bold)

### 6.6 Resume Components

#### ResumeUpload (`src/components/resumes/resume-upload.tsx`)
- Props: onUpload: (file) => void, isUploading: boolean
- Drag-and-drop zone
- File type validation (PDF, DOCX)
- Progress indicator

#### ResumeCard (`src/components/resumes/resume-card.tsx`)
- Props: resume: Resume, onDelete: () => void, onSetDefault: () => void
- Displays: name, version, default badge, ATS score, created date
- Actions: Delete, Set as Default

#### ResumeList (`src/components/resumes/resume-list.tsx`)
- Props: resumes: Resume[], isLoading: boolean
- Grid of ResumeCard components
- Empty state

#### ATSScore (`src/components/resumes/ats-score.tsx`)
- Props: score: number
- Visual score indicator (0-100)
- Color coding: <50 red, 50-70 yellow, 70-90 green, >90 blue

#### TailorResumeButton (`src/components/resumes/tailor-resume-button.tsx`)
- Props: resumeId: string, jobId: string, onTailor: () => void, isLoading: boolean
- Opens resume tailoring modal

### 6.7 Cover Letter Components

#### CoverLetterEditor (`src/components/cover-letters/cover-letter-editor.tsx`)
- Props: initialContent: string, onChange: (content) => void, isReadOnly: boolean
- Rich text editing
- Auto-save functionality

#### GenerateCoverLetter (`src/components/cover-letters/generate-cover-letter.tsx`)
- Props: jobId: string, onGenerate: () => void, generatedContent: string, isLoading: boolean
- Shows AI generation progress
- Preview generated content

### 6.8 Dashboard Components

#### StatsCard (`src/components/dashboard/stats-card.tsx`)
- Props: title: string, value: string | number, change?: string, icon?: ReactNode
- Metric display with optional trend

#### RecentApplications (`src/components/dashboard/recent-applications.tsx`)
- Props: applications: Application[]
- List of 5 most recent applications
- Status badges

#### QuickActions (`src/components/dashboard/quick-actions.tsx`)
- Discover Jobs button
- Upload Resume button
- View Saved Jobs button

---

## 7. Pages

### 7.1 Public Pages

#### Landing Page (`src/app/page.tsx`)
- Hero section with value proposition
- Feature highlights
- CTA to sign up
- Testimonials (optional)

#### Login Page (`src/app/(auth)/login/page.tsx`)
- OAuth buttons (GitHub, Google)
- Email/password form
- Link to signup
- Forgot password link

#### Signup Page (`src/app/(auth)/signup/page.tsx`)
- OAuth buttons
- Email/password form
- Target roles selector (checkboxes)
- Location input
- Remote only toggle

### 7.2 Dashboard Pages

#### Dashboard Home (`src/app/(dashboard)/dashboard/page.tsx`)
- Welcome message with user name
- Stats cards (Applications, Saved Jobs, This Week)
- Recent applications list
- Quick actions
- Upcoming follow-ups (if any)

#### Jobs Page (`src/app/(dashboard)/jobs/page.tsx`)
- JobFilters component
- JobList component
- Pagination
- Discover Jobs floating action button

#### Job Details Page (`src/app/(dashboard)/jobs/[id]/page.tsx`)
- JobDetails component
- Save/Apply actions
- Tailor Resume button

#### Discover Jobs Page (`src/app/(dashboard)/jobs/discover/page.tsx`)
- Source selection checkboxes
- Target roles selection
- Location input
- Discover button with progress
- Results preview

#### Applications Page (`src/app/(dashboard)/applications/page.tsx`)
- Application list
- Status filter tabs
- Application stats summary

#### Resumes Page (`src/app/(dashboard)/resumes/page.tsx`)
- ResumeList component
- Upload new resume button
- Empty state

#### Resume Upload Page (`src/app/(dashboard)/resumes/upload/page.tsx`)
- ResumeUpload component
- Name input
- Set as default toggle

#### Resume Details Page (`src/app/(dashboard)/resumes/[id]/page.tsx`)
- Resume preview
- Tailored versions list
- ATS score visualization

#### Settings Page (`src/app/(dashboard)/settings/page.tsx`)
- Profile form (name, email)
- Target roles multi-select
- Location input
- Remote only toggle
- Job sources selection
- Delete account

---

## 8. Services & Utilities

### 8.1 Supabase Client (`src/lib/supabase/client.ts`)

```typescript
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

### 8.2 Supabase Server (`src/lib/supabase/server.ts`)

```typescript
import { createServerClient, type CookieOptions } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return cookieStore.get(name)?.value
        },
        set(name: string, value: string, options: CookieOptions) {
          try {
            cookieStore.set({ name, value, ...options })
          } catch (error) {
            // Handle cookie errors
          }
        },
        remove(name: string, options: CookieOptions) {
          try {
            cookieStore.set({ name, value: '', ...options })
          } catch (error) {
            // Handle cookie errors
          }
        },
      },
    }
  )
}
```

### 8.3 Supabase Admin (`src/lib/supabase/admin.ts`)

```typescript
import { createClient } from '@supabase/supabase-js'
import { ServerRuntime } from 'next'

export function createAdminClient() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )
}
```

### 8.4 Utilities (`src/lib/utils.ts`)

```typescript
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

export function formatSalary(min?: number, max?: number, currency = 'USD') {
  if (!min && !max) return 'Salary not specified'
  const formatter = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
    maximumFractionDigits: 0
  })
  if (min && max) return `${formatter.format(min)} - ${formatter.format(max)}`
  if (min) return `From ${formatter.format(min)}`
  return `Up to ${formatter.format(max!)}`
}

export function formatDate(date: string | Date) {
  return new Date(date).toLocaleDateString('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric'
  })
}

export function timeAgo(date: string | Date) {
  const now = new Date()
  const then = new Date(date)
  const seconds = Math.floor((now.getTime() - then.getTime()) / 1000)
  
  const intervals = {
    year: 31536000,
    month: 2592000,
    week: 604800,
    day: 86400,
    hour: 3600,
    minute: 60
  }
  
  for (const [unit, secondsInUnit] of Object.entries(intervals)) {
    const interval = Math.floor(seconds / secondsInUnit)
    if (interval >= 1) {
      return `${interval} ${unit}${interval > 1 ? 's' : ''} ago`
    }
  }
  return 'Just now'
}
```

### 8.5 Constants (`src/lib/constants.ts`)

```typescript
export const JOB_SOURCES = ['linkedin', 'wellfound', 'otta', 'github', 'manual'] as const

export const JOB_STATUSES = ['new', 'saved', 'applied', 'archived'] as const

export const APPLICATION_STATUSES = [
  'pending',
  'applied',
  'viewed',
  'responded',
  'interview',
  'rejected',
  'accepted'
] as const

export const TARGET_ROLES = [
  'MLOps Engineer',
  'AI Platform Engineer',
  'SRE',
  'DevOps Engineer',
  'Solutions Engineer',
  'Software Engineer',
  'Data Engineer',
  'Machine Learning Engineer'
] as const

export const DEFAULT_TARGET_ROLES: string[] = [
  'MLOps Engineer',
  'AI Platform Engineer',
  'SRE',
  'DevOps',
  'Solutions Engineer'
]
```

---

## 9. State Management

### 9.1 Auth Store (`src/stores/auth-store.ts`)

Using Zustand for client-side auth state:

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface AuthState {
  user: any | null
  session: any | null
  isLoading: boolean
  setUser: (user: any) => void
  setSession: (session: any) => void
  setLoading: (loading: boolean) => void
  logout: () => void
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      session: null,
      isLoading: true,
      setUser: (user) => set({ user }),
      setSession: (session) => set({ session }),
      setLoading: (isLoading) => set({ isLoading }),
      logout: () => set({ user: null, session: null })
    }),
    {
      name: 'auth-storage'
    }
  )
)
```

### 9.2 Jobs Store (`src/stores/jobs-store.ts`)

```typescript
import { create } from 'zustand'
import type { Job } from '@/types/database'

interface JobsState {
  jobs: Job[]
  currentJob: Job | null
  isLoading: boolean
  filters: {
    status?: string
    source?: string
    search?: string
    remote?: boolean
  }
  pagination: {
    page: number
    total: number
    totalPages: number
  }
  setJobs: (jobs: Job[]) => void
  addJob: (job: Job) => void
  updateJob: (id: string, updates: Partial<Job>) => void
  removeJob: (id: string) => void
  setCurrentJob: (job: Job | null) => void
  setFilters: (filters: Partial<JobsState['filters']>) => void
  setPagination: (pagination: Partial<JobsState['pagination']>) => void
  setLoading: (loading: boolean) => void
}

export const useJobsStore = create<JobsState>((set) => ({
  jobs: [],
  currentJob: null,
  isLoading: false,
  filters: {},
  pagination: {
    page: 1,
    total: 0,
    totalPages: 0
  },
  setJobs: (jobs) => set({ jobs }),
  addJob: (job) => set((state) => ({ jobs: [job, ...state.jobs] })),
  updateJob: (id, updates) =>
    set((state) => ({
      jobs: state.jobs.map((j) => (j.id === id ? { ...j, ...updates } : j))
    })),
  removeJob: (id) =>
    set((state) => ({
      jobs: state.jobs.filter((j) => j.id !== id)
    })),
  setCurrentJob: (currentJob) => set({ currentJob }),
  setFilters: (filters) =>
    set((state) => ({ filters: { ...state.filters, ...filters } })),
  setPagination: (pagination) =>
    set((state) => ({ pagination: { ...state.pagination, ...pagination } })),
  setLoading: (isLoading) => set({ isLoading })
}))
```

### 9.3 UI Store (`src/stores/ui-store.ts`)

```typescript
import { create } from 'zustand'

interface UIState {
  sidebarOpen: boolean
  mobileMenuOpen: boolean
  jobDetailsOpen: boolean
  toastMessage: string | null
  toastType: 'success' | 'error' | 'info' | null
  toggleSidebar: () => void
  toggleMobileMenu: () => void
  setJobDetailsOpen: (open: boolean) => void
  showToast: (message: string, type: UIState['toastType']) => void
  hideToast: () => void
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: true,
  mobileMenuOpen: false,
  jobDetailsOpen: false,
  toastMessage: null,
  toastType: null,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  toggleMobileMenu: () =>
    set((state) => ({ mobileMenuOpen: !state.mobileMenuOpen })),
  setJobDetailsOpen: (jobDetailsOpen) => set({ jobDetailsOpen }),
  showToast: (toastMessage, toastType) => set({ toastMessage, toastType }),
  hideToast: () => set({ toastMessage: null, toastType: null })
}))
```

---

## 10. Authentication

### 10.1 Setup

1. **Supabase Auth Configuration:**
   - Go to Authentication > Providers in Supabase dashboard
   - Enable Email/Password
   - Enable GitHub (create OAuth app in GitHub Developer settings)
   - Enable Google (create OAuth app in Google Cloud Console)

2. **Environment Variables:**
```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

### 10.2 Auth Flow

1. **Sign Up:**
   - User fills form or clicks OAuth
   - Supabase creates auth user
   - Trigger creates profile in `public.profiles`
   - Return session to client

2. **Sign In:**
   - Validate credentials
   - Return session

3. **Session Management:**
   - Store in localStorage via Zustand persist
   - Refresh automatically via Supabase
   - Handle refresh token rotation

### 10.3 Protected Routes

Using middleware (`src/middleware.ts`):

```typescript
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({
    request
  })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return request.cookies.get(name)?.value
        },
        set(name: string, value: string, options: any) {
          request.cookies.set({
            name,
            value,
            ...options
          })
          supabaseResponse = NextResponse.next({
            request: {
              headers: request.headers
            }
          })
          supabaseResponse.cookies.set({
            name,
            value,
            ...options
          })
        },
        remove(name: string, options: any) {
          request.cookies.set({
            name,
            value: '',
            ...options
          })
          supabaseResponse = NextResponse.next({
            request: {
              headers: request.headers
            }
          })
          supabaseResponse.cookies.set({
            name,
            value: '',
            ...options
          })
        }
      }
    }
  )

  const { data: { user } } = await supabase.auth.getUser()

  // Protect dashboard routes
  if (request.nextUrl.pathname.startsWith('/dashboard') && !user) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Redirect to dashboard if already logged in
  if ((request.nextUrl.pathname === '/login' || request.nextUrl.pathname === '/signup') && user) {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }

  return supabaseResponse
}

export const config = {
  matcher: [
    '/dashboard/:path*',
    '/login',
    '/signup'
  ]
}
```

---

## 11. Job Discovery & Scraper

### 11.1 Scraper Services

#### LinkedIn Scraper (`src/services/scrapers/linkedin.ts`)

```typescript
interface LinkedInJob {
  jobId: string
  title: string
  company: string
  location: string
  remote: boolean
  salary?: { min: number; max: number; currency: string }
  description: string
  requirements: string[]
  url: string
  postedAt: string
}

export async function fetchLinkedInJobs(params: {
  keywords: string[]
  location: string
  remoteOnly: boolean
}): Promise<LinkedInJob[]> {
  // Note: LinkedIn requires authentication or scraping
  // For production, consider using LinkedIn API or a third-party service
  // This is a placeholder for the integration
  throw new Error('LinkedIn scraper not implemented - use API or third-party')
}
```

#### Wellfound Scraper (`src/services/scrapers/wellfound.ts`)

```typescript
interface WellfoundJob {
  id: string
  title: string
  company: string
  location: string
  remote: boolean
  salary?: { min: number; max: number; currency: string }
  description: string
  requirements: string[]
  url: string
  postedAt: string
}

export async function fetchWellfoundJobs(params: {
  keywords: string[]
  location: string
}): Promise<WellfoundJob[]> {
  const response = await fetch('https://wellfound.com/api/v1/jobs', {
    headers: {
      'Authorization': `Bearer ${process.env.WELLFOUND_API_KEY}`
    }
  })
  
  if (!response.ok) {
    throw new Error('Failed to fetch Wellfound jobs')
  }
  
  const data = await response.json()
  return data.jobs.map(transformWellfoundJob)
}

function transformWellfoundJob(raw: any): WellfoundJob {
  return {
    id: raw.id,
    title: raw.title,
    company: raw.company?.name,
    location: raw.location,
    remote: raw.remote === true,
    salary: raw.salary_min && raw.salary_max ? {
      min: raw.salary_min,
      max: raw.salary_max,
      currency: raw.salary_currency || 'USD'
    } : undefined,
    description: raw.description,
    requirements: raw.requirements || [],
    url: raw.url,
    postedAt: raw.published_at
  }
}
```

#### Otta Scraper (`src/services/scrapers/otta.ts`)

```typescript
interface OttaJob {
  id: string
  title: string
  company: string
  location: string
  remote: boolean
  salary?: { min: number; max: number; currency: string }
  description: string
  requirements: string[]
  url: string
  postedAt: string
}

export async function fetchOttaJobs(params: {
  keywords: string[]
  location: string
}): Promise<OttaJob[]> {
  const response = await fetch('https://api.otta.com/api/v1/jobs', {
    headers: {
      'Authorization': `Bearer ${process.env.OTTA_API_KEY}`
    }
  })
  
  if (!response.ok) {
    throw new Error('Failed to fetch Otta jobs')
  }
  
  const data = await response.json()
  return data.jobs.map(transformOttaJob)
}

function transformOttaJob(raw: any): OttaJob {
  return {
    id: raw.id,
    title: raw.title,
    company: raw.company?.name,
    location: raw.location,
    remote: raw.remote || false,
    salary: raw.compensation?.salary ? {
      min: raw.compensation.salary.min,
      max: raw.compensation.salary.max,
      currency: raw.compensation.salary.currency || 'USD'
    } : undefined,
    description: raw.description,
    requirements: raw.requirements || [],
    url: raw.url,
    postedAt: raw.published_at
  }
}
```

#### GitHub Jobs (`src/services/scrapers/github-jobs.ts`)

```typescript
interface GitHubJob {
  id: string
  title: string
  company: string
  location: string
  remote: boolean
  description: string
  requirements: string[]
  url: string
  postedAt: string
}

export async function fetchGitHubJobs(params: {
  keywords: string[]
  location: string
}): Promise<GitHubJob[]> {
  const { keywords, location } = params
  const query = encodeURIComponent(`${keywords.join(' ')} ${location}`)
  
  const response = await fetch(`https://jobs.github.com/positions.json?search=${query}`)
  
  if (!response.ok) {
    throw new Error('Failed to fetch GitHub jobs')
  }
  
  const data = await response.json()
  return data.map(transformGitHubJob)
}

function transformGitHubJob(raw: any): GitHubJob {
  return {
    id: raw.id,
    title: raw.title,
    company: raw.company,
    location: raw.location,
    remote: raw.location?.toLowerCase().includes('remote') || false,
    description: raw.description,
    requirements: [], // Extract from description if needed
    url: raw.url,
    postedAt: raw.created_at
  }
}
```

### 11.2 Job Discovery API (`src/app/api/jobs/discover/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase/server'
import { fetchWellfoundJobs } from '@/services/scrapers/wellfound'
import { fetchOttaJobs } from '@/services/scrapers/otta'
import { fetchGitHubJobs } from '@/services/scrapers/github-jobs'

export async function POST(request: NextRequest) {
  try {
    const supabase = createClient()
    
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    
    const body = await request.json()
    const { sources = ['wellfound', 'otta', 'github'], targetRoles, location, remoteOnly } = body
    
    let allJobs: any[] = []
    let duplicatesSkipped = 0
    let jobsSaved = 0
    
    // Fetch from each source
    for (const source of sources) {
      try {
        let jobs: any[] = []
        
        switch (source) {
          case 'wellfound':
            jobs = await fetchWellfoundJobs({ keywords: targetRoles, location })
            break
          case 'otta':
            jobs = await fetchOttaJobs({ keywords: targetRoles, location })
            break
          case 'github':
            jobs = await fetchGitHubJobs({ keywords: targetRoles, location })
            break
          // LinkedIn would require additional setup
        }
        
        allJobs = [...allJobs, ...jobs.map(j => ({ ...j, source }))]
      } catch (error) {
        console.error(`Error fetching from ${source}:`, error)
      }
    }
    
    // Save jobs to database (skip duplicates)
    for (const job of allJobs) {
      const { error: checkError } = await supabase
        .from('jobs')
        .select('id')
        .eq('user_id', user.id)
        .eq('source', job.source)
        .eq('source_id', job.id)
        .single()
      
      if (checkError) {
        // Job doesn't exist, insert it
        const { error: insertError } = await supabase
          .from('jobs')
          .insert({
            user_id: user.id,
            source: job.source,
            source_id: job.id,
            title: job.title,
            company: job.company,
            location: job.location,
            remote: job.remote,
            salary_min: job.salary?.min,
            salary_max: job.salary?.max,
            salary_currency: job.salary?.currency,
            description: job.description,
            requirements: job.requirements,
            url: job.url,
            posted_at: job.postedAt,
            status: 'new'
          })
        
        if (!insertError) {
          jobsSaved++
        }
      } else {
        duplicatesSkipped++
      }
    }
    
    return NextResponse.json({
      success: true,
      jobsFound: allJobs.length,
      jobsSaved,
      duplicatesSkipped,
      message: `Job discovery completed. Found ${jobsSaved} new jobs.`
    })
    
  } catch (error) {
    console.error('Job discovery error:', error)
    return NextResponse.json(
      { error: 'Failed to discover jobs' },
      { status: 500 }
    )
  }
}
```

---

## 12. AI Resume Tailoring

### 12.1 OpenAI Service (`src/services/openai.ts`)

```typescript
import OpenAI from 'openai'

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
})

export interface TailorResumeParams {
  resumeContent: string
  jobTitle: string
  jobCompany: string
  jobDescription: string
  jobRequirements: string[]
}

export interface TailorResumeResult {
  tailoredContent: string
  matchedKeywords: string[]
  missingKeywords: string[]
  improvementsSuggested: string[]
  atsScoreBefore: number
  atsScoreAfter: number
}

export async function tailorResume(params: TailorResumeParams): Promise<TailorResumeResult> {
  const { resumeContent, jobTitle, jobCompany, jobDescription, jobRequirements } = params
  
  const prompt = `
You are an expert resume tailor. Your task is to modify a resume to better match a job posting.

## Job Information
- Title: ${jobTitle}
- Company: ${jobCompany}
- Description: ${jobDescription}
- Requirements: ${jobRequirements.join(', ')}

## Current Resume
${resumeContent}

## Instructions
1. Analyze the job requirements and identify keywords that are present/missing in the resume
2. Modify the resume content to better highlight relevant experience and skills
3. Keep the resume structure professional and ATS-friendly
4. Do not fabricate experiences - only emphasize existing relevant experience

## Output Format (JSON)
{
  "tailoredContent": "The modified resume text",
  "matchedKeywords": ["keyword1", "keyword2", ...],
  "missingKeywords": ["keyword3", "keyword4", ...],
  "improvementsSuggested": ["suggestion1", "suggestion2", ...],
  "atsScoreBefore": <estimated score 0-100>,
  "atsScoreAfter": <estimated score 0-100>
}

Provide ONLY the JSON output, no additional text.
`

  const response = await openai.chat.completions.create({
    model: 'gpt-4-turbo-preview',
    messages: [
      {
        role: 'system',
        content: 'You are an expert resume writer and ATS optimization specialist.'
      },
      {
        role: 'user',
        content: prompt
      }
    ],
    response_format: { type: 'json_object' },
    temperature: 0.3
  })

  const result = JSON.parse(response.choices[0].message.content!)
  
  return {
    tailoredContent: result.tailoredContent,
    matchedKeywords: result.matchedKeywords,
    missingKeywords: result.missingKeywords,
    improvementsSuggested: result.improvementsSuggested,
    atsScoreBefore: result.atsScoreBefore,
    atsScoreAfter: result.atsScoreAfter
  }
}

export async function generateCoverLetter(params: {
  resumeContent: string
  jobTitle: string
  jobCompany: string
  jobDescription: string
  tone?: 'professional' | 'friendly' | 'enthusiastic'
}): Promise<string> {
  const { resumeContent, jobTitle, jobCompany, jobDescription, tone = 'professional' } = params
  
  const tone guidance = {
    professional: 'Maintain a professional, formal tone.',
    friendly: 'Use a warm, friendly tone while remaining professional.',
    enthusiastic: 'Express genuine excitement and enthusiasm for the role.'
  }
  
  const prompt = `
Write a cover letter for a job application.

## Job Information
- Title: ${jobTitle}
- Company: ${jobCompany}
- Description: ${jobDescription}

## Candidate Resume
${resumeContent}

## Tone
${tone guidance[tone]}

## Instructions
1. Write a compelling cover letter that highlights relevant experience
2. Address the hiring manager professionally
3. Show enthusiasm for the role and company
4. Keep it to 3-4 paragraphs
5. Do not fabricate experiences

Provide ONLY the cover letter text, no formatting or explanations.
`

  const response = await openai.chat.completions.create({
    model: 'gpt-4-turbo-preview',
    messages: [
      {
        role: 'system',
        content: 'You are an expert cover letter writer.'
      },
      {
        role: 'user',
        content: prompt
      }
    ],
    temperature: 0.5
  })

  return response.choices[0].message.content!
}
```

### 12.2 Resume Tailoring API (`src/app/api/resumes/tailor/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase/server'
import { tailorResume } from '@/services/openai'

export async function POST(request: NextRequest) {
  try {
    const supabase = createClient()
    
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    
    const body = await request.json()
    const { resumeId, jobId } = body
    
    if (!resumeId || !jobId) {
      return      return NextResponse.json(
        { error: 'resumeId and jobId are required' },
        { status: 400 }
      )
    }
    
    // Get resume
    const { data: resume, error: resumeError } = await supabase
      .from('resumes')
      .select('*')
      .eq('id', resumeId)
      .eq('user_id', user.id)
      .single()
    
    if (resumeError || !resume) {
      return NextResponse.json({ error: 'Resume not found' }, { status: 404 })
    }
    
    // Get job
    const { data: job, error: jobError } = await supabase
      .from('jobs')
      .select('*')
      .eq('id', jobId)
      .eq('user_id', user.id)
      .single()
    
    if (jobError || !job) {
      return NextResponse.json({ error: 'Job not found' }, { status: 404 })
    }
    
    // Call OpenAI to tailor resume
    const result = await tailorResume({
      resumeContent: resume.content || '',
      jobTitle: job.title,
      jobCompany: job.company,
      jobDescription: job.description || '',
      jobRequirements: job.requirements || []
    })
    
    // Save tailored resume
    const { data: tailoredResume, error: insertError } = await supabase
      .from('tailored_resumes')
      .insert({
        user_id: user.id,
        job_id: jobId,
        original_resume_id: resumeId,
        content: result.tailoredContent,
        matched_keywords: result.matchedKeywords,
        missing_keywords: result.missingKeywords,
        improvements_suggested: result.improvementsSuggested,
        ats_score_before: result.atsScoreBefore,
        ats_score_after: result.atsScoreAfter
      })
      .select()
      .single()
    
    if (insertError) {
      console.error('Error saving tailored resume:', insertError)
      return NextResponse.json(
        { error: 'Failed to save tailored resume' },
        { status: 500 }
      )
    }
    
    // Track analytics event
    await supabase.from('analytics_events').insert({
      user_id: user.id,
      event_type: 'resume_tailored',
      event_data: { job_id: jobId, resume_id: resumeId }
    })
    
    return NextResponse.json({
      tailoredResume: {
        id: tailoredResume.id,
        content: tailoredResume.content,
        matchedKeywords: tailoredResume.matched_keywords,
        missingKeywords: tailoredResume.missing_keywords,
        improvementsSuggested: tailoredResume.improvements_suggested,
        atsScoreBefore: tailoredResume.ats_score_before,
        atsScoreAfter: tailoredResume.ats_score_after
      }
    })
    
  } catch (error) {
    console.error('Resume tailoring error:', error)
    return NextResponse.json(
      { error: 'Failed to tailor resume' },
      { status: 500 }
    )
  }
}
```

---

## 13. Environment Variables Reference

### Required Variables

```env
# Supabase (Required)
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# OpenAI (Required for AI features)
OPENAI_API_KEY=sk-your-api-key
```

### Optional Variables

```env
# Job APIs (Optional)
WELLFOUND_API_KEY=your-wellfound-key
OTTA_API_KEY=your-otta-key

# OAuth Providers (Optional - for additional auth methods)
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

### .env.local.example

Create this file in the project root:

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# OpenAI
OPENAI_API_KEY=sk-your-api-key

# Job APIs (Optional)
WELLFOUND_API_KEY=
OTTA_API_KEY=

# OAuth
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
```

---

## 14. Error Handling Strategy

### 14.1 API Error Handling

All API routes follow consistent error handling:

```typescript
export async function handler(request: NextRequest) {
  try {
    // 1. Validate authentication
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    
    // 2. Validate request body
    const body = await request.json()
    const validated = schema.parse(body)
    
    // 3. Execute business logic
    const result = await doSomething(validated)
    
    // 4. Return success
    return NextResponse.json(result)
    
  } catch (error) {
    // Handle validation errors
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      )
    }
    
    // Handle known errors
    if (error instanceof KnownError) {
      return NextResponse.json(
        { error: error.message },
        { status: error.statusCode }
      )
    }
    
    // Log unexpected errors
    console.error('Unexpected error:', error)
    
    // Return generic error
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}
```

### 14.2 Frontend Error Handling

```typescript
// Using React Query
const { data, error, isLoading } = useQuery({
  queryKey: ['jobs'],
  queryFn: () => fetchJobs(),
})

if (error) {
  toast.error(error.message)
}

// Using useEffect
useEffect(() => {
  async function fetchData() {
    try {
      const response = await apiCall()
      setData(response)
    } catch (error) {
      if (error instanceof ApiError) {
        setError(error.message)
      } else {
        setError('An unexpected error occurred')
      }
    }
  }
  fetchData()
}, [])
```

### 14.3 Error Types

| Error Type | HTTP Status | Example |
|------------|-------------|---------|
| Unauthorized | 401 | Not logged in |
| Forbidden | 403 | Accessing another user's data |
| Not Found | 404 | Job/resume doesn't exist |
| Validation Error | 400 | Invalid input data |
| Rate Limited | 429 | Too many requests |
| Server Error | 500 | Internal error |

---

## 15. Testing Approach

### 15.1 Unit Tests

**Location:** `src/**/*.test.ts` or `src/**/*.test.tsx`

**Coverage:**
- Utility functions
- Validation schemas
- Component rendering
- Store actions

**Example:**
```typescript
// src/lib/utils.test.ts
import { describe, it, expect } from 'vitest'
import { formatSalary, formatDate, timeAgo } from './utils'

describe('formatSalary', () => {
  it('formats salary range', () => {
    expect(formatSalary(100000, 150000)).toBe('$100,000 - $150,000')
  })
  
  it('handles no salary', () => {
    expect(formatSalary()).toBe('Salary not specified')
  })
})

describe('timeAgo', () => {
  it('returns "Just now" for recent dates', () => {
    const now = new Date().toISOString()
    expect(timeAgo(now)).toBe('Just now')
  })
  
  it('returns hours ago', () => {
    const twoHoursAgo = new Date(Date.now() - 2 * 60 * 60 * 1000).toISOString()
    expect(timeAgo(twoHoursAgo)).toBe('2 hours ago')
  })
})
```

### 15.2 Integration Tests

**Location:** `tests/**/*.test.ts`

**Coverage:**
- API routes
- Database operations
- Authentication flows

**Example:**
```typescript
// tests/api/jobs.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import request from 'supertest'
import { app } from '@/app'

describe('GET /api/jobs', () => {
  let token: string
  
  beforeAll(async () => {
    // Setup: create test user and get token
    token = await getTestToken()
  })
  
  it('returns 401 without auth', async () => {
    const response = await request(app).get('/api/jobs')
    expect(response.status).toBe(401)
  })
  
  it('returns jobs for authenticated user', async () => {
    const response = await request(app)
      .get('/api/jobs')
      .set('Authorization', `Bearer ${token}`)
    
    expect(response.status).toBe(200)
    expect(Array.isArray(response.body.jobs)).toBe(true)
  })
})
```

### 15.3 E2E Tests (Optional)

**Location:** `e2e/**/*.test.ts`

**Tools:** Playwright

**Example:**
```typescript
// e2e/job-discovery.test.ts
import { test, expect } from '@playwright/test'

test('user can discover and save jobs', async ({ page }) => {
  // 1. Login
  await page.goto('/login')
  await page.fill('[name="email"]', 'test@example.com')
  await page.fill('[name="password"]', 'password')
  await page.click('button[type="submit"]')
  
  // 2. Navigate to discover
  await page.click('text=Discover Jobs')
  
  // 3. Start discovery
  await page.click('button:has-text("Discover Jobs")')
  await page.waitForSelector('text=Job discovery completed')
  
  // 4. Save a job
  await page.click('button:has-text("Save"):first')
  await page.waitForSelector('text=Job saved')
  
  // 5. Verify in saved jobs
  await page.click('text=Saved')
  await expect(page.locator('.job-card')).toHaveCount(1)
})
```

### 15.4 Testing Commands

```bash
# Run unit tests
npm run test

# Run with coverage
npm run test:coverage

# Run integration tests
npm run test:integration

# Run E2E tests
npm run test:e2e

# Run all tests
npm run test:all
```

---

## 16. Phase-Based Implementation Guide

### Phase 1: Project Setup (Day 1)

**Goals:**
- [ ] Create Next.js project with TypeScript and Tailwind
- [ ] Set up Supabase project and run migrations
- [ ] Configure environment variables
- [ ] Set up Supabase client and auth
- [ ] Create basic layout and navigation

**Files to Create:**
- `src/lib/supabase/client.ts`
- `src/lib/supabase/server.ts`
- `src/lib/utils.ts`
- `src/middleware.ts`
- `src/app/layout.tsx`
- `src/app/(auth)/layout.tsx`
- `src/app/(dashboard)/layout.tsx`

**Testing:**
- Verify Supabase connection
- Verify auth flow works
- Verify middleware protects routes

### Phase 2: Authentication (Day 2)

**Goals:**
- [ ] Create login/signup pages
- [ ] Implement OAuth (GitHub, Google)
- [ ] Create profile management
- [ ] Set up protected routes

**Files to Create:**
- `src/app/(auth)/login/page.tsx`
- `src/app/(auth)/signup/page.tsx`
- `src/components/auth/auth-form.tsx`
- `src/components/auth/oauth-buttons.tsx`
- `src/app/api/auth/signup/route.ts`
- `src/app/api/auth/signin/route.ts`
- `src/app/api/auth/me/route.ts`
- `src/app/api/auth/profile/route.ts`

**Testing:**
- Test email/password signup
- Test OAuth flow
- Test protected routes redirect

### Phase 3: Job Management (Day 3-4)

**Goals:**
- [ ] Create jobs API routes
- [ ] Build jobs listing page
- [ ] Implement job filters
- [ ] Create job details view
- [ ] Implement job saving

**Files to Create:**
- `src/app/api/jobs/route.ts`
- `src/app/api/jobs/[id]/route.ts`
- `src/app/api/jobs/[id]/save/route.ts`
- `src/app/(dashboard)/jobs/page.tsx`
- `src/app/(dashboard)/jobs/[id]/page.tsx`
- `src/components/jobs/job-card.tsx`
- `src/components/jobs/job-list.tsx`
- `src/components/jobs/job-filters.tsx`
- `src/components/jobs/job-details.tsx`

**Testing:**
- List jobs with filters
- View job details
- Save job to list

### Phase 4: Job Discovery (Day 5-6)

**Goals:**
- [ ] Set up scraper services
- [ ] Create job discovery API
- [ ] Build discover page UI
- [ ] Implement match scoring

**Files to Create:**
- `src/services/scrapers/wellfound.ts`
- `src/services/scrapers/otta.ts`
- `src/services/scrapers/github-jobs.ts`
- `src/app/api/jobs/discover/route.ts`
- `src/app/(dashboard)/jobs/discover/page.tsx`
- `src/components/jobs/discover-jobs-button.tsx`

**Testing:**
- Fetch jobs from each source
- Save new jobs to database
- Test match score calculation

### Phase 5: Resume Management (Day 7-8)

**Goals:**
- [ ] Create resume upload functionality
- [ ] Build resumes listing page
- [ ] Implement resume storage
- [ ] Create ATS scoring

**Files to Create:**
- `src/app/api/resumes/route.ts`
- `src/app/api/resumes/upload/route.ts`
- `src/app/(dashboard)/resumes/page.tsx`
- `src/app/(dashboard)/resumes/upload/page.tsx`
- `src/components/resumes/resume-upload.tsx`
- `src/components/resumes/resume-card.tsx`
- `src/components/resumes/resume-list.tsx`
- `src/components/resumes/ats-score.tsx`

**Testing:**
- Upload PDF/DOCX resume
- List user resumes
- Display ATS score

### Phase 6: AI Resume Tailoring (Day 9-10)

**Goals:**
- [ ] Set up OpenAI integration
- [ ] Create resume tailoring API
- [ ] Build tailoring UI
- [ ] Display keyword analysis

**Files to Create:**
- `src/services/openai.ts`
- `src/app/api/resumes/tailor/route.ts`
- `src/app/(dashboard)/resumes/[id]/page.tsx`
- `src/components/resumes/tailor-resume-button.tsx`

**Testing:**
- Tailor resume for specific job
- Verify matched/missing keywords
- Test AI-generated content quality

### Phase 7: Cover Letters (Day 11)

**Goals:**
- [ ] Create cover letter generation API
- [ ] Build cover letter UI
- [ ] Store generated cover letters

**Files to Create:**
- `src/app/api/cover-letters/route.ts`
- `src/app/api/cover-letters/generate/route.ts`
- `src/components/cover-letters/cover-letter-editor.tsx`
- `src/components/cover-letters/generate-cover-letter.tsx`

**Testing:**
- Generate cover letter for job
- Edit generated content
- Save cover letter

### Phase 8: Applications (Day 12)

**Goals:**
- [ ] Create applications API
- [ ] Build applications listing
- [ ] Implement application creation
- [ ] Add one-click apply URL

**Files to Create:**
- `src/app/api/applications/route.ts`
- `src/app/api/applications/[id]/route.ts`
- `src/app/(dashboard)/applications/page.tsx`
- `src/components/applications/application-card.tsx`
- `src/components/applications/application-form.tsx`
- `src/components/applications/application-status-badge.tsx`

**Testing:**
- Create application
- View application list
- Test one-click apply URL

### Phase 9: Dashboard & Analytics (Day 13-14)

**Goals:**
- [ ] Create dashboard home page
- [ ] Implement stats cards
- [ ] Add analytics API
- [ ] Build recent activity

**Files to Create:**
- `src/app/api/analytics/route.ts`
- `src/app/(dashboard)/dashboard/page.tsx`
- `src/components/dashboard/stats-card.tsx`
- `src/components/dashboard/recent-applications.tsx`
- `src/components/dashboard/quick-actions.tsx`

**Testing:**
- View dashboard stats
- Navigate to recent items
- Test quick actions

### Phase 10: Polish & Deployment (Day 15)

**Goals:**
- [ ] Add error handling
- [ ] Implement loading states
- [ ] Add toast notifications
- [ ] Set up deployment
- [ ] Test production build

**Files to Create/Update:**
- `src/components/ui/toast.tsx`
- `src/stores/ui-store.ts`
- Update all pages with loading states

**Testing:**
- Test all error scenarios
- Verify loading states
- Test production build

---

## Quick Start Checklist

- [ ] Created Next.js project with TypeScript and Tailwind
- [ ] Set up Supabase project and ran migrations
- [ ] Configured environment variables in `.env.local`
- [ ] Implemented authentication (signup, login, OAuth)
- [ ] Created job management (CRUD, listing, filters)
- [ ] Implemented job discovery (scrapers)
- [ ] Created resume upload and storage
- [ ] Implemented AI resume tailoring
- [ ] Created cover letter generation
- [ ] Built applications tracking
- [ ] Created dashboard with stats
- [ ] Tested production build
- [ ] Deployed to Vercel/hosting

---

## Next Steps

1. **Review this implementation plan**
2. **Set up development environment**
3. **Begin Phase 1 implementation**
4. **Test each feature as it's built**
5. **Deploy and iterate**

---

*Implementation Plan Version: 1.0*  
*Last Updated: 2026-03-09*
