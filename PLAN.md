# Rallyify Plan

Status legend: ⬜ not started · 🟨 in progress · ✅ done (verified by tester) · ⏸ blocked

Workflow per task: dev implements and self-verifies → tester independently verifies against REQUIREMENTS.md → team lead marks done and updates this file. A task is only ✅ after tester sign-off.

## Phase 1 — Foundation ⬜

Goal: both packages scaffolded, DB schema live locally, tests runnable, hello-world deployable.

| # | Task | Acceptance criteria |
|---|------|---------------------|
| 1.1 | Scaffold `api/`: Hono + TypeScript + wrangler config + vitest (workers pool) | `wrangler dev` serves a health endpoint; `npm test` runs a passing smoke test |
| 1.2 | D1 schema + local init script (`npm run db:init`) | Schema from ARCHITECTURE.md applies cleanly; tables queryable locally |
| 1.3 | Auth middleware (bearer secret for host routes) | Host routes 401 without token, 200 with; public routes open |
| 1.4 | Scaffold `web/`: React + Vite + Tailwind + router + typed API client stub | `vite dev` renders a landing placeholder; API client compiles with typed methods matching ARCHITECTURE.md |
| 1.5 | DEVELOPMENT.md (build/test/run instructions for both packages) | Tester can set up and run everything from the doc alone |

## Phase 2 — Event management (API-first) ⬜

Goal: full event lifecycle via API — the Claude-as-host interface (REQUIREMENTS MVP #6–7).

| # | Task | Acceptance criteria |
|---|------|---------------------|
| 2.1 | Event CRUD endpoints (create/get/list/update/delete) with invite + admin token generation | All endpoints work via curl; validation rejects missing title; tokens unique |
| 2.2 | Slot endpoints (add/edit/delete), date-only and timed slots | Timezone rules from ARCHITECTURE.md hold: date-only never shifted; timed slots carry origin tz |
| 2.3 | Theme storage on event (JSON field, defaults, PATCH support) | Theme set/updated via API; unknown keys rejected; defaults returned when unset |

## Phase 3 — Responder flow ⬜

Goal: an invitee with a link can respond on their phone (REQUIREMENTS MVP #11–14, #18–19).

| # | Task | Acceptance criteria |
|---|------|---------------------|
| 3.1 | Public event GET (invite-token scoped, public fields only) | Admin token and other invitees' emails never leak |
| 3.2 | Response submission endpoint (name required, email optional, yes/no/maybe per slot, edit until finalized/deadline) | Duplicate name updates the existing response; submissions blocked after finalize/deadline |
| 3.3 | Responder page: event display, slot list in viewer's timezone, availability picker | Times render in viewer tz with origin tz shown; date-only slots show no time; mobile-usable |
| 3.4 | Confirmation page after responding | Shows what was submitted; edit path works |

## Phase 4 — Host UI & decision ⬜

Goal: host can create events in a browser and close the loop (REQUIREMENTS MVP #1–5, #15–17).

| # | Task | Acceptance criteria |
|---|------|---------------------|
| 4.1 | Host dashboard + event create form (title, description, slots, deadline, invitee-slots toggle stored but inert) | Event created in UI matches one created via API |
| 4.2 | Host event detail: response grid with per-slot yes/maybe/no counts | Grid matches responses submitted via API |
| 4.3 | Best-slot highlighting (most yes, fewest no) | Ties highlighted together; obvious visually |
| 4.4 | Finalize action: pick slot, event status → finalized, responder page shows confirmed time | Responses locked after finalize; responder page shows decision |

## Phase 5 — Theming ⬜

Goal: per-event look and feel (REQUIREMENTS MVP #8–10).

| # | Task | Acceptance criteria |
|---|------|---------------------|
| 5.1 | Theme → CSS custom property injection on responder page | All theme fields visibly applied; unset fields fall back to defaults |
| 5.2 | Theme editor in host event detail | Round-trips through API; live preview or clear apply |
| 5.3 | Extensibility check: add one new theme field end-to-end as proof | Diff touches only theme type/default, one UI control, one CSS var reference |

## Phase 6 — Deploy & polish ⬜

Goal: live on the internet, presentable.

| # | Task | Acceptance criteria |
|---|------|---------------------|
| 6.1 | Production D1 database + Workers deploy + secret | API reachable at workers.dev URL (custom domain when purchased) |
| 6.2 | Pages deploy wired to API | Full loop works in production: create via API → respond on phone → finalize |
| 6.3 | Landing page + empty states + deadline enforcement pass | No dead ends; deadline blocks responses with a friendly message |
| 6.4 | README.md (user-reviewed before this repo is shared) | Reviewed and approved by user |

## Deferred (Phase 2+ of REQUIREMENTS.md)

Notifications, reminders, calendar export/import, invitee-proposed slots (data model ships in MVP; flow deferred), host accounts, and everything in REQUIREMENTS.md Phase 3+.

## Decisions log

- 2026-07-03: Cloudflare Workers + D1 + Pages chosen over Vercel/Fly (consistency with Clarmory, serverless, no idle cost).
- 2026-07-03: Single dev agent + tester agent + lead; models Sonnet 4.6 for both (revisit if dev struggles).
- 2026-07-03: Domain purchase (rallyify.com) deferred; workers.dev / pages.dev URLs until then.
