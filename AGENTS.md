# AGENTS.md - Combined Architecture Guide (Lovable TS/JS/React/Vite/Supabase)
Use this guide for three phases: before first build, after first build, and continuous audit/fix/iterate.

## 1) Start Before Building a New Lovable App
- Define domain boundaries first: auth, monitoring/events, admin CMS, and shared UI.
- Decide source of truth per layer: Supabase for persistent data, Zustand stores for client session/UI state, component state for local ephemeral interactions.
- Set folder contracts early (`src/pages`, `src/components`, `src/stores`, `src/integrations/supabase`) and avoid cross-layer imports that bypass public module entry points.
- Design app_config + RLS for editable content before UI implementation so CMS features do not require schema rewrites later.
- Prefer typed contracts first (`types.ts`, schema-aligned query result types, discriminated unions for UI modes) before wiring pages.

## 2) Layout + Navigation (Current Repo Contract)
Use only the unified layout in `src/components/layouts/`:
- `UnifiedSidebar.tsx` (single sidebar for all auth states)
- `UnifiedLayout.tsx` (sidebar + page content wrapper)
- `index.ts` (exports)

Default page pattern:
- import from `@/components/layouts`
- wrap page content in `UnifiedLayout`

`UnifiedSidebar` nav states:
- Logged out: Home, Status, Changelog, Terms (`/tos`), Sign-up, Sign-in
- Logged in user: Dashboard, Monitors, Profile, Status, Changelog, Terms, Sign out
- Logged in admin: Dashboard, Monitors, Profile, Admin, Status, Changelog, Terms, Sign out

Footer rules:
- `src/components/Footer.tsx` reads CMS HTML from `app_config.footer_html`
- Footer is embedded in `UnifiedSidebar.tsx`
- Do not render standalone `<Footer />` in pages

Deprecated (do not reintroduce):
- `PublicNavbar.tsx`
- custom per-page layout wrappers
- standalone page footers

## 3) CMS Content Pattern (`app_config`)
CMS values are JSON strings in `app_config`:
- `tos_html` -> Terms content (`TosPage.tsx`)
- `footer_html` -> Footer content (`Footer.tsx`)

Data handling:
- read `value` by `key`, then `JSON.parse(value)`
- write with `upsert({ key, value: JSON.stringify(html) })`
- writes are admin-only

## 4) Engineering Principles
**DRY:** In this stack, each behavior should have one canonical implementation: one layout system, one auth state source (`useAuthStore()`), one CMS storage pattern (`app_config` JSON), and one API/data-access path per feature; when similar logic appears in two pages/components, extract it to shared hooks/services only after confirming the shared abstraction preserves readability and does not hide Supabase query intent.

**S - Single Responsibility:** Keep React components focused on rendering and user interaction, move Supabase I/O into dedicated functions/hooks, and keep stores responsible only for state transitions; if a file mixes rendering, remote fetch orchestration, and domain rules, split it by concern.

**O - Open/Closed:** Extend behavior via composition (new page sections, store actions, feature flags, or layout variants) instead of editing stable core flows; new features should plug into existing contracts (`UnifiedLayout`, store APIs, typed query helpers) without rewriting existing consumers.

**L - Liskov Substitution:** Any swapped implementation (mock service, alternate store adapter, or layout variant) must preserve the same input/output contract and side-effect expectations so calling code does not branch on implementation type.

**I - Interface Segregation:** Expose small task-focused interfaces (for example, `AuthSessionSlice`, `CmsConfigClient`, `MonitorEventsReader`) rather than broad objects with unrelated methods, so pages and hooks depend only on the capabilities they actually use.

**D - Dependency Inversion:** Depend on abstractions at the feature boundary (typed service functions, injected clients, and pure utility interfaces), not concrete Supabase or browser primitives inside UI components, so testability and replacement remain straightforward.

**State hygiene:** Keep local UI state local, shared cross-page state in scoped Zustand slices, and persisted truth in Supabase; represent async workflows with explicit status fields (`idle | loading | success | error`), reset transient state on route/identity changes, and ban duplicate writable copies of the same entity across component state and store state.

## 5) When to Switch to FSM Pattern
Use this matrix for React UI flow decisions:

| Situation | Recommended Pattern |
|---|---|
| 2 or fewer independent toggles, no invalid combinations | `useState` booleans |
| 3+ mutually exclusive modes, invalid combinations possible | discriminated union + reducer |
| Multi-step async flow (retry/cancel/timeout/backoff), guard conditions, or role-dependent transitions | explicit FSM (state + event + transition map) |

Short rule: if you need more than one boolean to represent a single mode, or you can create impossible states, move to an FSM-style model.

## 6) State Management Defaults
Auth store (`src/stores/auth-store.ts`):
- `useAuthStore()` is source of truth
- `isAdmin()` for admin checks
- `signOut()` for logout

Events store (`src/stores/events-store.ts`):
- bounded by `MAX_EVENTS = 500`
- used for dashboard event feed

## 7) Theme/Branding (Future-Ready)
Before adding branding/layout logic, inspect `// Future:` comments in:
- `UnifiedSidebar.tsx` (logo/product/colors)
- `UnifiedLayout.tsx` (layout variants)

Planned components (build only when requested):
- `TopNavLayout.tsx`
- `MobileDrawer.tsx`
- `LayoutPicker.tsx`

## 8) Supabase RLS Requirements
`app_config` intent must remain:
- public read allowed
- writes allowed only for authenticated admins (`is_admin(auth.uid())`)

Verify RLS before adding new `app_config` keys.

## 9) Post First Build Checklist
1. Confirm critical user journeys (auth, dashboard, monitor lifecycle, admin CMS edit/publish) pass in local and preview environments.
2. Run verification: `bun run typecheck`, `bun run lint`, `bun run test`, `supabase db diff`.
3. Validate state boundaries: no duplicate writable entity state across local state/store/Supabase.
4. Validate security: RLS enforced for all writable tables; admin-only writes verified.
5. Validate resilience: loading/error/empty states exist for each remote data surface.
6. Update `CHANGELOG.md` with version bump and major architecture deltas.

## 10) Continuous Audit -> Fix -> Iterate Loop
1. Audit: inspect runtime errors, slow queries, flaky flows, UX dead-ends, and schema/policy drift.
2. Prioritize: rank by user impact, security risk, and frequency.
3. Fix in small diffs: one pattern/file cluster at a time, preserving existing contracts.
4. Verify: rerun typecheck/lint/tests/db diff and re-test touched journeys.
5. Iterate: capture lessons in this guide and repeat on a regular cadence (at least every release cycle).

## 11) Key Paths
- Layouts: `src/components/layouts/`
- Pages: `src/pages/`
- UI: `src/components/ui/`
- Stores: `src/stores/`
- Supabase client: `src/integrations/supabase/`
- Edge functions: `supabase/functions/`
