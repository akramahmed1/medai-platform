# SKILLS.md — Verified Patterns Library
**Project:** Medical AI Enablement Platform
**Rule:** If a pattern exists here, USE IT — never re-invent. If a needed pattern is missing, propose it here first (in a PR description), get approval, then use it. Improvising around these patterns is a constitution violation.

---

## S1. Migration Workflow

Every schema change is a new file: `supabase/migrations/NNNN_description.sql`
- Files are sequential, immutable once merged to main
- Never edit a merged migration file — write a new one
- Never use the Supabase dashboard for schema changes
- Every migration touching a table with RLS includes its policies in the same file
- Run locally: `npx supabase db push` against the dev project only

---

## S2. Supabase Client Pattern (use exactly — no ad-hoc clients)

```typescript
// lib/supabase/server.ts — for server components, server actions, API routes
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export function createClient() {
  const cookieStore = cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { get: (n) => cookieStore.get(n)?.value } }
  )
}

// lib/supabase/client.ts — for client components only
import { createBrowserClient } from '@supabase/ssr'
export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}

// lib/supabase/service.ts — service-role client
// ONLY imported by: certificate generation action, /api/verify, pnpm seed
// Every other import is a PR-blocking violation
import { createClient as createSupabaseClient } from '@supabase/supabase-js'
export const serviceClient = createSupabaseClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)
```

---

## S3. Signed URL Pattern (video + certificates)

```typescript
// In a server action — never in a client component
async function getSignedVideoUrl(lessonId: string): Promise<string> {
  const supabase = createClient()  // server client — S2
  // 1. Verify session
  const { data: { session } } = await supabase.auth.getSession()
  if (!session) throw new Error('Unauthenticated')
  // 2. Verify enrollment (RLS handles org scoping)
  const lesson = await supabase.from('lessons').select('video_path, module_id').eq('id', lessonId).single()
  if (!lesson.data?.video_path) throw new Error('No video')
  // 3. Generate signed URL (60-minute expiry)
  const { data } = await supabase.storage.from('videos').createSignedUrl(lesson.data.video_path, 3600)
  return data!.signedUrl
}
// Client never receives bucket paths. Buckets are private. Zero public storage policies.
```

---

## S4. Theming Pattern

```typescript
// app/layout.tsx — ThemeProvider reads org + program from profile
// Sets CSS variables: --brand-primary, --brand-accent, --brand-logo-url
// All components use: className="bg-[var(--brand-primary)]" or Tailwind CSS var syntax

// FORBIDDEN — zero hardcoded colors in components:
// ❌ className="bg-[#1B3A5C]"
// ❌ style={{ color: '#0E7C7B' }}
// ✅ className="bg-[var(--brand-primary)]"
```

Grep gate before every push (S12): `grep -ri "#[0-9a-fA-F]\{6\}" src/components` → 0 results

---

## S5. AI Assistant Pattern

```typescript
// app/api/assistant/route.ts
import { streamText } from 'ai'
import { anthropic } from '@ai-sdk/anthropic'  // via Vercel AI SDK — never @anthropic-ai/sdk directly

export async function POST(req: Request) {
  // 1. Rate limit check (20/hr/user)
  // 2. Validate: max 1000 chars, max 30 turns in conversation
  // 3. FTS retrieval: org-scoped kb_documents, top 5
  const context = await retrieveKBContext(orgId, userMessage)
  // 4. Stream
  const result = await streamText({
    model: anthropic('claude-sonnet-4-6'),  // swap to AI SDK provider pattern at pilot
    system: SYSTEM_PROMPT,  // verbatim from CLAUDE.md §8 — do not paraphrase
    messages: [...last6Turns, { role: 'user', content: userMessage + '\n\nContext:\n' + context }],
    maxTokens: 500
  })
  // 5. Persist both turns to assistant_messages (org-scoped WITH CHECK enforced by RLS)
  return result.toDataStreamResponse()
}
```

---

## S6. RLS Policy Templates (copy verbatim — change table name only)

```sql
-- (a) Learner owns rows: read only
create policy {t}_select on {t} for select using (user_id = auth.uid());

-- (b) Learner owns rows: insert WITH BOTH bindings (never omit org_id check)
create policy {t}_insert on {t} for insert
  with check (user_id = auth.uid() and org_id = get_user_org_id());

-- (c) Learner owns rows: update WITH BOTH bindings
create policy {t}_update on {t} for update
  using (user_id = auth.uid() and org_id = get_user_org_id());

-- (d) Org admin: full access scoped to own org
create policy {t}_admin on {t} for all
  using (org_id = get_user_org_id() and is_admin());

-- (e) Program-scoped supervisor (tables with program_id)
create policy {t}_supervisor on {t} for all using (
  supervisor_user_id = auth.uid()
  and org_id = get_user_org_id()
  and program_id = get_user_program_id()
);

-- (f) Platform operator — ONLY via SECURITY DEFINER function
create policy {t}_platform on {t} for all using (is_platform_operator());
```

**BANNED forms** (each caused a real security bug in this project):
```sql
-- ❌ Cross-tenant admin read leak:
create policy x on t for select using (org_id = get_user_org_id() OR is_admin());

-- ❌ Profile self-escalation (bare update lets learner set role='admin'):
create policy x on profiles for update using (user_id = auth.uid());
-- ✅ Use column-level grants instead (see CLAUDE.md §4)

-- ❌ Platform operator raw subquery (breaks silently when platform_operators has RLS):
create policy x on t for all using (auth.uid() in (select user_id from platform_operators));
-- ✅ Use is_platform_operator() SECURITY DEFINER function

-- ❌ Insert without WITH CHECK:
create policy x on enrollments for all using (user_id = auth.uid());
-- ✅ Split into select + insert with check + update
```

---

## S7. Eight-Test RLS Suite (run before every merge)

```typescript
// tests/rls.test.ts — runs as authenticated personas, NOT service role

// T1: Cross-vendor isolation
// As learner-vendorA: SELECT from courses WHERE org_id = vendorB.id → 0 rows

// T2: Cross-program supervisor
// As supervisor-program1: SELECT from profiles WHERE program_id = program2.id → 0 rows

// T3: Org insert pollution
// As learner: INSERT into enrollments (user_id=self, org_id=foreignOrg) → RLS error

// T4: Audit log isolation
// As org admin: SELECT from audit_logs → 0 rows (operator-only)

// T5: Platform operator sees all
// As operator: SELECT from organizations → count = all vendors

// T6: Assessment cross-program
// As supervisor-program1: INSERT into assessment_results (program_id=program2) → RLS error

// T7: Role self-escalation
// As learner: UPDATE profiles SET role='admin' WHERE user_id=self → column grant error

// T8: Org self-reassignment
// As learner: UPDATE profiles SET org_id=foreignOrg WHERE user_id=self → column grant error

// ALL tests use authenticated JWT personas — never service role for assertions
```

---

## S8. Certificate Generation Pattern

```typescript
// Server action — called on course completion, runs transactionally
async function generateCertificate(userId: string, courseId: string) {
  const service = serviceClient  // from lib/supabase/service.ts only
  // 1. Generate cert_uid: CRT-XXXX-XXXX (Crockford base32, 8 random chars)
  // 2. Render PDF with @react-pdf/renderer:
  //    - Org logo (via CSS var resolved at render time)
  //    - Learner name, course title, issued date
  //    - Expiry date (issued + 24 months)
  //    - QR code → /verify/[certUid]
  //    - Cert UID printed below QR
  // 3. Upload to 'certificates' bucket (private): path = `{orgId}/{certUid}.pdf`
  // 4. Insert certificates row
  // 5. Update enrollments.status = 'completed', completed_at = now()
  // Verification route /verify/[certUid]:
  //   - Service client lookup (bypasses RLS)
  //   - Returns ONLY: {valid, learner_name, course_title, issued_at, expires_at}
  //   - Invalid uid → {valid: false} — same shape, never 500
}
```

---

## S9. Component Structure Rules

```
Every async component ships three states in the same PR:
  Loading:  <Skeleton /> from shadcn/ui — matches layout of loaded state
  Empty:    Designed empty state with icon + message (not just blank)
  Error:    <ErrorBoundary> + user-friendly message + retry where applicable

Mobile-first: design at 360px, verify at 768px and 1280px.
3G throttle test: Chrome DevTools → Network → Slow 3G before closing any video-related PR.
No new shadcn components without running: npx shadcn@latest add {component}
```

---

## S10. Environment Variables (both Supabase projects)

```bash
# .env.local (dev) — never commit to git
NEXT_PUBLIC_SUPABASE_URL=https://[dev-project-ref].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[dev-anon-key]
SUPABASE_SERVICE_ROLE_KEY=[dev-service-role-key]  # server-only

# Sentry
NEXT_PUBLIC_SENTRY_DSN=[dsn]

# AI (server-only — never expose to client)
# Vercel AI SDK picks this up automatically:
ANTHROPIC_API_KEY=[key]

# Vercel preview password (set in Vercel dashboard)
# VERCEL_PROTECTION_BYPASS=[token]
```

---

## S11. Evidence Protocol (claiming "done")

| What shipped | Required evidence |
|---|---|
| Schema / RLS change | Paste 8-test output (all green) |
| UI component | Screenshot at 360px and 1280px |
| API route | curl command + response JSON |
| AI assistant change | `pnpm eval:assistant` output (all 7 prompts pass/fail) |
| Certificate generation | Screenshot of generated PDF + /verify URL returning valid JSON |
| CSV export | Screenshot of file opened in Excel (UTF-8, no garbled chars) |
| Any PR | Neutrality grep output (0 results) |

"It should work", "I tested it locally", "looks good" — none of these count as evidence.

---

## S12. Git & PR Conventions

```
PR title:    Task N: Short description  (e.g. "Task 1: Scaffold + schema + RLS + seed")
Branch:      task-N-short-name
PR template:
  ## Plan
  What: [what this PR does]
  Why: [which acceptance criteria from CLAUDE.md §10]
  Patterns: [which SKILLS.md patterns used]

  ## Changes
  - [file path] — [what changed]

  ## Evidence
  [paste evidence per S11]

  ## Acceptance criteria checklist
  - [ ] [criteria from CLAUDE.md §10 for this task]

  ## New dependencies (if any)
  - [package] — [one-line justification]
```

One PR per task. No force-push to main. PRs reviewed against acceptance criteria before merge.

