# ENV_SETUP.md — Environment & Repository Setup Guide
**For Akram: follow these steps exactly in order before giving Claude Code access.**

---

## Step 1: Create GitHub Repository

1. Go to github.com → New repository
2. Name: `medai-platform` (or your preferred name)
3. Private: YES
4. Do NOT initialize with README (you'll push from local)
5. Copy the repository URL

## Step 2: Create Local Project Folder

```bash
# On your machine:
mkdir medai-platform
cd medai-platform

# Download all project files into this folder:
# - CLAUDE.md
# - SKILLS.md
# - ROADMAP.md
# - ENV_SETUP.md (this file)
# Place them in the root of the folder

# Initialize git:
git init
git add .
git commit -m "Initial: project constitution and skills"
git remote add origin [your-github-url]
git push -u origin main
```

## Step 3: Create Two Supabase Projects

Go to supabase.com → New project (do this twice):

**Project 1 — Development:**
- Name: `medai-platform-dev`
- Database password: save it securely
- Region: US East (or closest to you)

**Project 2 — Production:**
- Name: `medai-platform-prod`
- Database password: save it securely (different from dev)
- Region: same as dev for now

For each project, go to Project Settings → API and copy:
- Project URL
- anon/public key
- service_role key (keep this secret — never commit)

## Step 4: Install Supabase CLI

```bash
# Mac:
brew install supabase/tap/supabase

# Or via npm:
npm install -g supabase

# Link to your dev project:
supabase login
supabase link --project-ref [dev-project-ref]
```

## Step 5: Create .env.local File

Create a file called `.env.local` in the project root (this file is git-ignored):

```bash
# Development Supabase
NEXT_PUBLIC_SUPABASE_URL=https://[dev-project-ref].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[dev-anon-key]
SUPABASE_SERVICE_ROLE_KEY=[dev-service-role-key]

# AI (server-only — Vercel AI SDK picks this up automatically)
ANTHROPIC_API_KEY=[your-anthropic-api-key]

# Sentry (get free DSN from sentry.io after creating a Next.js project)
NEXT_PUBLIC_SENTRY_DSN=[your-sentry-dsn]
```

Make sure `.env.local` is in `.gitignore` (it will be automatically with Next.js).

## Step 6: Create Vercel Project

1. Go to vercel.com → New project → Import from GitHub
2. Select `medai-platform`
3. Framework: Next.js (auto-detected)
4. Add environment variables (same as .env.local above)
5. Add a second environment for prod (different Supabase project refs)
6. Enable Vercel Authentication (password protection) for the preview URL

## Step 7: Give Claude Code Access

1. Open Claude Code
2. Open the `medai-platform` folder
3. Say: "Read CLAUDE.md and SKILLS.md. We are starting Task 1. Post your plan before writing any code."

That is all. Claude Code reads the constitution and skills file and follows the engineering loop from §18.

---

## What Claude Code will build in Task 1:

- Next.js 15 project scaffold with TypeScript and Tailwind
- `supabase/migrations/001_core_schema.sql` — all tables
- `supabase/migrations/002_rls_policies.sql` — all RLS policies + column grants
- `supabase/migrations/003_audit_triggers.sql` — audit triggers
- `scripts/seed.ts` (`pnpm seed`) — idempotent seed script
- Sentry wired in `instrumentation.ts`
- `lib/supabase/server.ts`, `client.ts`, `service.ts`
- Verified by: `pnpm seed` runs clean + all 8 RLS tests pass

**Do not run migrations on the prod project until the demo is complete.**
