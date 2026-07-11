# ROADMAP.md — Pilot & Beyond
**These items are NEVER built in the demo. Any request to build them during the demo phase is a scope violation. Log it here and proceed with the frozen demo scope.**

## Pilot Phase (after demo feedback → signed SOW)
- [ ] Background job layer (Inngest or Trigger.dev) for video processing, batch certs, notifications
- [ ] Supabase PITR backups + restore drill
- [ ] Content authoring approval UI (AI pipeline drafts; human approves before publish)
- [ ] Playwright E2E tests in CI (golden path)
- [ ] `pnpm eval:assistant` scoring as automated CI gate
- [ ] Rate limiting on auth endpoints + /verify (middleware or Upstash)
- [ ] Security headers (CSP, HSTS, X-Frame-Options)
- [ ] Resend email: enrollment confirmation, certificate issued, expiry reminder
- [ ] Bulk CSV user import
- [ ] UI i18n (Hindi, Tamil, Bahasa, Thai) — assistant language mirroring ships in demo; full UI translation is pilot
- [ ] REST API: training records export endpoint (for ministry integrations)
- [ ] Webhooks: course completion events
- [ ] WhatsApp notifications (completion, expiry reminders) via WATI
- [ ] Per-program hosting region selection (data residency for Singapore, India)
- [ ] Data Processing Agreement (DPA) template finalized and signed before prod data

## Rollout Phase
- [ ] Subdomain-per-tenant routing (vendor.academy.com → config, not code)
- [ ] SSO / SAML for enterprise identity providers
- [ ] PWA + offline module caching for field environments (low-connectivity screening vans)
- [ ] Train-the-trainer cohort tier (cascade training model)
- [ ] Certificate expiry recertification engine (auto-flag + delta course)
- [ ] Per-program billing and payments (Stripe)
- [ ] SCORM / LTI 1.3 import (for existing content libraries)
- [ ] FTS → pgvector semantic retrieval (trigger: KB > 500 chunks or first non-English course)

## Scale Phase (12+ months)
- [ ] Per-tenant LLM provider config + BYOK (AWS Bedrock, Azure OpenAI, Google Vertex, regional inference)
- [ ] Voice assistant (STT/TTS in regional languages: Hindi, Tamil, Swahili, Bahasa) for field operators
- [ ] AI content pipeline UI (upload raw video → AI segments → human approval → publish)
- [ ] In-product guided training overlay (WalkMe/Pendo pattern — a separate product, not an LMS feature)
- [ ] Accredited CE credit integrations (requires accreditation body partnerships)
- [ ] Multi-vendor platform (additional vendor orgs with separate contracts)
- [ ] Public certificate verification API (for hospital HR systems)
- [ ] Co-published implementation research with anchor vendor
- [ ] Advanced analytics: time-to-competency trends, cross-program benchmarks
- [ ] Video CDN migration (Cloudflare R2 or Bunny) for zero-egress-cost bandwidth at scale
