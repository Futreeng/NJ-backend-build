# Phase 1 Plan: Core (Database, Auth, Public Pages, Admin Shell)

**Status:** Ready to execute | Date: 2026-10-16

---

## Objectives

1. Supabase local database with schema, RLS, triggers, helper functions.
2. Next.js app with TypeScript strict, Tailwind with design tokens, light/dark mode.
3. Public pages reading real data: Home, Calendar, Locals.
4. Admin shell: secretary form, approval queue, emergency controls, activity log.
5. Authorization guards enforcing approval gate and role boundaries.
6. Cache behavior: pause triggers visibility within 10 seconds.
7. Security: Zod validation, CSP, HSTS, no service-role in client.
8. Tests: RLS (14.1.1-11), XSS, security headers, e2e, axe, Lighthouse.
9. Kill-switch drill performed and timed.

---

## Architecture Decisions (To Record in docs/DECISIONS/)

1. **Next.js app location:** In `/web` directory (avoids collision with App Router /app folder).
2. **Supabase setup:** Local CLI with Docker (docker-compose).
3. **Database first:** Schema migrations before server code, tests written with each policy.
4. **Auth approach:** Supabase Auth (email/password) + invite-only flow, MFA required for council_admin, re-auth for sensitive actions.
5. **Component library:** Radix UI for headless dialogs/menus/tabs.
6. **Rate limiting:** Postgres-based (avoid extra vendor for Phase 1).
7. **Testing:** Vitest (unit), Playwright (e2e), SQL tests for RLS written immediately after each policy.
8. **Security:** CSP and headers in Next.js skeleton (days 3-4, not deferred).

---

## Task List (Priority Order)

### Phase 1A: Database and Migrations (Days 1-2)

1. [ ] Initialize Supabase project locally with `supabase start`.
2. [ ] Create migration: enums (app_role, post_type, publish_status).
3. [ ] Create migration: locals table, seed nine locals.
4. [ ] Create migration: settings table (global_pause, member_voice_enabled, alerts_enabled).
5. [ ] Create migration: profiles table (users, roles, local_id).
6. [ ] Create migration: posts table with indexes and constraints.
7. [ ] Create helper functions: is_council(), my_local(), is_paused() (all SECURITY DEFINER).
8. [ ] Create activity_log table (append-only).
9. [ ] Write and run SQL RLS tests for 14.1.1 (secretary cannot publish).
10. [ ] Test: secretary cannot read cross-local drafts (14.1.2-3).
11. [ ] Test: anonymous cannot read pending (14.1.4).

### Phase 1B: RLS Policies and Triggers with Tests (Days 2-3)

Each policy and trigger includes its test immediately. A policy is not done until its test passes.

1. [ ] Enable RLS on all tables (default deny).
2. [ ] RLS policy + test: posts anonymous SELECT (published, not paused) (14.1.4).
3. [ ] RLS policy + test: posts secretary SELECT/INSERT/UPDATE (own local only) (14.1.2-3).
4. [ ] RLS policy + test: posts council SELECT/UPDATE (all).
5. [ ] RLS policy + test: activity_log (no INSERT/UPDATE/DELETE from API, council SELECT only) (14.1.8).
6. [ ] RLS policy + test: settings (SELECT via view, UPDATE via function only).
7. [ ] RLS policy + test: profiles (user reads own, council reads all).
8. [ ] Trigger + test: status_transition_guard (reject invalid transitions).
9. [ ] Trigger + test: on published, set approved_by and published_at server-side.
10. [ ] Trigger + test: audit every INSERT/status change to activity_log.
11. [ ] Trigger + test: updated_at on posts, profiles.
12. [ ] Test: secretary cannot publish (14.1.1).
13. [ ] Test: global_pause makes anon SELECT return zero (14.1.5-6).
14. [ ] Rough kill-switch drill: pause all, verify public returns zero rows (preliminary, not timed).

### Phase 1C: Next.js Setup, Auth, Security Headers (Days 3-4)

1. [ ] Initialize Next.js (App Router, TypeScript strict, Tailwind).
2. [ ] Install: @supabase/ssr, @supabase/auth-helpers-nextjs, zod, radix-ui, class-variance-authority.
3. [ ] Configure Tailwind with CSS variables (section 6 tokens).
4. [ ] Implement light/dark mode toggle (prefers-color-scheme + data-theme).
5. [ ] Add security headers to middleware: strict CSP, HSTS, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, frame-ancestors none.
6. [ ] Create lib/auth/guards.ts: requireCouncil(), requireSecretaryFor(localId).
7. [ ] Create lib/auth/invite.ts: create invite (72-hour single-use token), accept invite (set password, optional MFA).
8. [ ] Implement MFA enrollment: required for council_admin during invite acceptance.
9. [ ] Implement re-authentication: required before pause toggle, role change, delete operations.
10. [ ] Create lib/supabase/client.ts (browser) and server.ts (server-only).
11. [ ] Create lib/validators/: schema files for posts, profiles, etc. (Zod).
12. [ ] Set up environment: NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY (no service-role in client).
13. [ ] Build rate limit helper (login: 5/15min per IP).
14. [ ] Set up GitHub Actions workflow: typecheck, lint, unit tests, RLS tests, dependency audit, gitleaks secret scan.
15. [ ] Test: build fails if service-role key appears in client bundle.

### Phase 1D: Public Pages (Days 4-5)

1. [ ] Create /app/(public)/page.tsx (Home): hero, filters, feed, sidebar.
2. [ ] Create /app/(public)/calendar/page.tsx: events grouped by month, local filter.
3. [ ] Create /app/(public)/locals/page.tsx: nine cards with update counts.
4. [ ] Create /app/(public)/locals/[id]/page.tsx: local feed detail.
5. [ ] Component: PostCard (date box, badges, "Arrived via", Add to Calendar).
6. [ ] Component: StatTile (locals, updates, events).
7. [ ] Component: EventList (upcoming six events in sidebar).
8. [ ] Query real data from posts (published only, not paused).
9. [ ] Query real data from meetings (sample seed + Local 819 real dates).
10. [ ] Search and filter (by local chip, by type chip).
11. [ ] Test: Lighthouse 90+ on Home (performance, accessibility, best practices, SEO).
12. [ ] Test: axe zero serious/critical violations.
13. [ ] Test: 200% zoom, no horizontal scroll on 375px.

### Phase 1E: Admin Shell (Days 5-6)

1. [ ] Create /app/(admin)/login: email/password form, rate limit, generic errors.
2. [ ] Create /app/(admin)/(authenticated)/page.tsx: shell layout with tabs.
3. [ ] Tab: Secretary view: "Post an update" form (local auto-filled, type, date, title, body).
4. [ ] Tab: Council view: approval queue (list pending posts, Approve/Reject buttons).
5. [ ] Tab: Council view: published posts (list published, Unpublish button).
6. [ ] Tab: Council view: emergency controls (red border, Pause ALL, pause per-local, toggle Member Voice, toggle Alerts).
7. [ ] Tab: Council view: activity log (filterable list, export as CSV planned).
8. [ ] Tab: Council view: people (invite form, list users, deactivate button).
9. [ ] Component: Modal (approve form with reject reason, alert preview with text/email mockup).
10. [ ] Server action: POST /api/posts (secretary submit, validate, set status=pending).
11. [ ] Server action: POST /api/approve (council only, transition to published, set approved_by/published_at).
12. [ ] Server action: POST /api/reject (council only, transition to rejected, store reason).
13. [ ] Server action: POST /api/pause-all (council only, set settings.global_pause, cache invalidation).
14. [ ] Server action: POST /api/pause-local (council only, set locals.paused per local).
15. [ ] Server action: POST /api/unpublish (council only, transition to unpublished).
16. [ ] Test: secretary cannot approve (14.1.1).
17. [ ] Test: secretary cannot change status beyond pending (14.1.3).
18. [ ] Test: council can do all (14.1 suite).

### Phase 1F: Cache and Kill-Switch Behavior (Days 6-7)

1. [ ] Implement ISR or on-demand revalidation for public pages.
2. [ ] Set TTL: 30 seconds max for public pages (or on-demand with revalidateTag).
3. [ ] Add cache invalidation tag: 'settings' (pause/resume triggers revalidateTag).
4. [ ] Add cache invalidation tag: 'posts-published' (approve/unpublish triggers revalidateTag).
5. [ ] Test: pause all, fresh browser within 10s shows empty (14.1.5-6).
6. [ ] Test: resume, fresh browser shows posts return.
7. [ ] Perform timed official kill-switch drill (record in docs/RUNBOOK.md with timestamp, participants, time taken).

### Phase 1G: Security Hardening (Days 7-8)

1. [ ] Add Zod validation at every server boundary (posts, auth, etc.).
2. [ ] Add security headers: Content-Security-Policy (strict), HSTS, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, frame-ancestors none.
3. [ ] Test: security header compliance (curl or automated checker).
4. [ ] Test: XSS payload corpus (section 14.2): render in titles, bodies, no script execution.
5. [ ] Test: CSRF protection (SameSite cookies, framework protections).
6. [ ] Audit: no service-role key in client bundle (build-time check via lint or test).
7. [ ] Add build check: fail if service-role key found in client code.

### Phase 1H: Testing Suite (Days 8-9)

1. [ ] Unit tests (Vitest): validators (Zod schemas).
2. [ ] Unit tests: auth guards (requireCouncil, requireSecretaryFor).
3. [ ] Unit tests: rate limit helpers.
4. [ ] SQL tests (pgTAP or Supabase): RLS policies (14.1.1-11).
5. [ ] E2E tests (Playwright): secretary submits, council approves, post appears public.
6. [ ] E2E tests: council pauses, public empty, resumes, content returns.
7. [ ] E2E tests: filters and search work.
8. [ ] Accessibility (axe): zero serious/critical violations.
9. [ ] Performance (Lighthouse CI): 90+ on Home.
10. [ ] Coverage: 80%+ on lib/, 100% on auth guards.

### Phase 1I: Documentation and Handoff (Days 9-10)

1. [ ] Update README.md: local dev setup for Windows (Supabase CLI, Docker, Node).
2. [ ] Create docs/PHASE_1_SETUP.md: step-by-step local setup guide.
3. [ ] Update docs/ARCHITECTURE.md with implementation details.
4. [ ] Update docs/THREAT_MODEL.md with verification status per threat.
5. [ ] Update docs/RUNBOOK.md: kill-switch drill result (time, participants, screenshots).
6. [ ] Create docs/DECISIONS/001-*.md for each architecture decision.
7. [ ] Write PHASE_1 completion summary: what works, test results, incomplete items.

---

## Testing Checklist

**RLS Tests (14.1):**
- [ ] 14.1.1: Secretary cannot publish (status transition blocked).
- [ ] 14.1.2-3: Secretary cannot read/edit cross-local drafts.
- [ ] 14.1.4: Anonymous cannot read pending/draft/rejected.
- [ ] 14.1.5-6: Global pause returns zero rows, cache clears within 10s.
- [ ] 14.1.8: Activity log immutable (no UPDATE/DELETE).
- [ ] 14.1.10: Deactivated user loses access immediately.

**Security Tests (14.2):**
- [ ] XSS payload corpus (titles, bodies): no execution.
- [ ] CSRF: cross-origin form blocked.
- [ ] Security headers: CSP, HSTS, etc. present and correct.

**E2E Tests:**
- [ ] Secretary submits → pending.
- [ ] Council approves → published → public.
- [ ] Council unpublishes → public empty.
- [ ] Pause all → public empty within 10s.
- [ ] Resume → content returns.
- [ ] Filters work (local chips, type chips, search).

**Quality Gates:**
- [ ] Axe: zero serious/critical.
- [ ] Lighthouse: 90+ on Home, 80+ on Calendar/Locals.
- [ ] Type check: `tsc --strict` clean.
- [ ] Lint: no errors.
- [ ] Coverage: 80%+ on lib/, 100% on auth.

---

## Dependencies (to verify available or install)

- Node.js 20+ (check with `node -v`)
- pnpm (install with `npm install -g pnpm`)
- Docker Desktop (for Supabase local)
- Supabase CLI (`pnpm add -D @supabase/cli`)
- Next.js 15+ (latest stable)
- React 19+
- TypeScript 5.5+
- Tailwind CSS 4+
- Zod latest
- Vitest latest
- Playwright latest
- Radix UI (select components)

---

## Known Unknowns / Assumptions for Phase 1

1. **Supabase local performance:** assuming Docker works smoothly on Windows. If issues, may need troubleshooting.
2. **Rate limiting implementation:** using Postgres COUNT/timestamp approach (can switch to Redis later if needed).
3. **Cache revalidation:** using Next.js ISR/revalidateTag. If issues, fall back to shorter TTLs.
4. **MFA in Phase 1:** OPTIONAL for Phase 1 (mentioned in spec, but can defer to Phase 2 if time-constrained).
5. **Email/SMS:** not in scope for Phase 1 (Phase 3+), so no email provider setup needed.

---

## Milestones

- **End of Day 2:** Database and locals seeded, migrations committed.
- **STOP AFTER DAY 3:** End of 1A and 1B. Report: schema, all policies, test output, anything assumed. Wait for go before 1C.
- **End of Day 4:** Next.js skeleton with auth, security headers, MFA, re-auth, invite flow.
- **End of Day 5:** Public pages displaying real data from database.
- **End of Day 6:** Admin shell complete, emergency controls working, rough kill-switch drill passed.
- **End of Day 9:** All tests passing, timed kill-switch drill completed.
- **End of Day 10:** Documentation updated, handoff ready.

---

## Estimated Timeline

**10 working days** (Phase 1A-I, assuming no major blockers and good Windows/Docker setup).

---

## Next Steps (After Plan Approval)

1. Set up Supabase local environment (docker-compose).
2. Initialize Next.js app structure.
3. Begin Phase 1A (database migrations).
4. Commit each day's work with tests.
5. Report blockers or lessons learned to docs/LESSONS.md daily.
6. At end of Phase 1: full test report, kill-switch drill time, demo walkthrough, Phase 2 plan.

---

**Waiting for approval to start Phase 1A.**
