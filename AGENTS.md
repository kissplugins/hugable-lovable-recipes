# AGENTS.md - Checklist-Driven Architecture Guide (Lovable TS/JS/React/Vite/Supabase)
Use this guide as an execution checklist for three phases: pre-build, post-first-build, and continuous audit/fix/iterate.

## 0) How to Use This Doc
- Treat each section as a pass/fail checklist.
- Do not mark items done without code evidence (files, tests, policies, or logs).
- Follow order: pre-build -> build contract -> post-build -> continuous loop.

## 1) Pre-Build Checklist (Before First Feature)
- [ ] Define domain boundaries: auth, monitoring/events, admin CMS, shared UI.
- [ ] Define source of truth: Supabase for persisted data, Zustand for shared client state, component state for local ephemeral UI.
- [ ] Freeze folder contracts: `src/pages`, `src/components`, `src/stores`, `src/integrations/supabase`.
- [ ] Design `app_config` keys and RLS before building CMS-driven pages.
- [ ] Define typed contracts first (query result types, union states, service interfaces).

## 2) Build Contract Checklist (Current Repo Rules)
- [ ] All pages use `UnifiedLayout` from `src/components/layouts/`.
- [ ] `UnifiedSidebar.tsx` remains the single sidebar for all auth states.
- [ ] `UnifiedSidebar` nav states remain:
  - logged out: Home, Status, Changelog, Terms (`/tos`), Sign-up, Sign-in
  - logged in user: Dashboard, Monitors, Profile, Status, Changelog, Terms, Sign out
  - logged in admin: Dashboard, Monitors, Profile, Admin, Status, Changelog, Terms, Sign out
- [ ] Footer remains embedded in `UnifiedSidebar.tsx`.
- [ ] Standalone `<Footer />` is not rendered in pages.
- [ ] Do not reintroduce deprecated patterns: `PublicNavbar.tsx`, custom per-page wrappers, standalone page footers.

## 3) CMS Content Checklist (`app_config`) (Retained)
- [ ] CMS values are JSON strings in `app_config`.
- [ ] Keys remain:
  - `tos_html` -> Terms content (`TosPage.tsx`)
  - `footer_html` -> Footer content (`Footer.tsx`)
- [ ] Read pattern: select `value` by `key`, then `JSON.parse(value)`.
- [ ] Write pattern: `upsert({ key, value: JSON.stringify(html) })`.
- [ ] Writes are admin-only.

## 4) State Management Defaults Checklist
- [ ] Auth source of truth is `useAuthStore()` in `src/stores/auth-store.ts`.
- [ ] Admin checks use `isAdmin()`.
- [ ] Logout uses `signOut()`.
- [ ] Events feed uses `src/stores/events-store.ts` with `MAX_EVENTS = 500`.
- [ ] No duplicated writable entity state across component state + store + Supabase.

## 5) Design Rules (Required Paragraphs)
**DRY:** In this stack, each behavior must have one canonical implementation: one layout system, one auth state source (`useAuthStore()`), one CMS storage pattern (`app_config` JSON), and one data-access path per feature; when duplicate logic appears in multiple pages/components, extract only after confirming the abstraction preserves readability and does not hide Supabase query intent.

**S - Single Responsibility:** Keep React components focused on rendering and user interaction, move Supabase I/O into service functions/hooks, and keep stores responsible for state transitions; split files that mix view rendering, remote orchestration, and domain rules.

**O - Open/Closed:** Extend behavior through composition (feature modules, store actions, layout variants, feature flags) instead of rewriting stable flows; new work should plug into existing contracts without breaking current consumers.

**L - Liskov Substitution:** Any replaceable implementation (mock service, alternate adapter, layout variant) must preserve the same input/output and side-effect expectations so call sites never branch on concrete type.

**I - Interface Segregation:** Expose small task-specific interfaces (for example auth session, CMS config client, monitor event reader) so pages/hooks depend only on the methods they use.

**D - Dependency Inversion:** Depend on feature-boundary abstractions (typed services/interfaces) rather than concrete Supabase/browser primitives inside UI components, improving testability and replacement.

**State hygiene:** Keep local UI state local, shared cross-page state in scoped Zustand slices, and persisted truth in Supabase; model async workflows with explicit status unions (`idle | loading | success | error`), reset transient state on route/identity changes, and avoid duplicate writable copies of the same entity.

## 6) FSM Decision Matrix + Trigger Checklist
| Situation | Recommended Pattern |
|---|---|
| 2 or fewer independent toggles, no invalid combinations | `useState` booleans |
| 3+ mutually exclusive modes, invalid combinations possible | discriminated union + reducer |
| multi-step async flow (retry/cancel/timeout/backoff), guards, role-dependent transitions | explicit FSM (state + event + transition map) |

Switch trigger checklist:
- [ ] More than one boolean is needed to represent a single UI mode.
- [ ] Impossible states can be represented accidentally.
- [ ] Transition rules depend on role, async outcome, or retries/timeouts.

## 7) Observability + Error Contract Checklist (New Required Item)
- [ ] Every async boundary (Supabase call, auth transition, edge function call) emits `start/success/failure` telemetry.
- [ ] Telemetry includes: `feature`, `requestId`, `durationMs`, and `errorCode` on failures.
- [ ] A shared app error shape is enforced across UI/store/server boundaries.
- [ ] Raw errors are normalized at boundaries before reaching UI components.
- [ ] User-facing error handling maps to contract categories (`validation`, `auth`, `permission`, `not_found`, `conflict`, `network`, `db`, `unknown`).

Recommended contract:
```ts
type AppError = {
  code: string;
  message: string;
  category: 'validation' | 'auth' | 'permission' | 'not_found' | 'conflict' | 'network' | 'db' | 'unknown';
  retryable: boolean;
  httpStatus?: number;
  requestId?: string;
  details?: Record<string, unknown>;
};

type Result<T> = { ok: true; data: T } | { ok: false; error: AppError };
```

## 8) Post First Build Checklist
- [ ] Critical journeys pass in local + preview (auth, dashboard, monitor lifecycle, admin CMS edit/publish).
- [ ] Verification commands pass: `bun run typecheck`, `bun run lint`, `bun run test`, `supabase db diff`.
- [ ] Security checks pass: RLS on writable tables, admin-only writes verified.
- [ ] Resilience checks pass: loading/error/empty states exist for each remote surface.
- [ ] `Observability + Error Contract Checklist` (Section 7) is fully passed.
- [ ] `CHANGELOG.md` is updated with version bump and architecture deltas.

## 9) Continuous Audit -> Fix -> Iterate Checklist
- [ ] Audit runtime signals: errors, slow queries, flaky journeys, UX dead-ends, schema/policy drift.
- [ ] Prioritize by user impact, security risk, and recurrence.
- [ ] Fix in small diffs (one pattern/file cluster at a time).
- [ ] Re-verify with typecheck/lint/tests/db diff and re-test touched journeys.
- [ ] Record learnings by updating this document.
- [ ] Repeat at least once per release cycle.

## 10) Supabase RLS Requirements Checklist
- [ ] `app_config` public read is enabled.
- [ ] `app_config` writes are restricted to authenticated admins (`is_admin(auth.uid())`).
- [ ] RLS policies are verified before adding new `app_config` keys.

## 11) Theme/Branding (Future-Ready) Checklist
- [ ] Check `// Future:` comments before adding branding/layout logic.
- [ ] `UnifiedSidebar.tsx` is the extension point for logo/product/colors.
- [ ] `UnifiedLayout.tsx` is the extension point for layout variants.
- [ ] Planned components (`TopNavLayout.tsx`, `MobileDrawer.tsx`, `LayoutPicker.tsx`) are added only when explicitly requested.

## 12) Key Paths
- Layouts: `src/components/layouts/`
- Pages: `src/pages/`
- UI: `src/components/ui/`
- Stores: `src/stores/`
- Supabase client: `src/integrations/supabase/`
- Edge functions: `supabase/functions/`
