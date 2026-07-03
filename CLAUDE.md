# Rallyify

Multiparty scheduling app: a host proposes event times, invitees respond with availability via a shareable link, the host finalizes. See MISSION_STATEMENT.md.

## Read before working
- **REQUIREMENTS.md** — what to build, by phase. The source of truth for acceptance.
- **ARCHITECTURE.md** — stack, API contract, DB schema, timezone strategy, theming design.
- **PLAN.md** — current phase, task breakdown, and status.

## Team roles
- **Team lead** (main session): coordinates work, maintains docs, talks to the user. Does not implement.
- **dev**: sole implementer, owns `api/` and `web/`.
- **tester**: acceptance gate; runs and verifies dev's work against REQUIREMENTS.md. Does not write code.

## Project conventions
- Stack: Cloudflare Workers (Hono, TypeScript) + D1 in `api/`; React + Vite (TypeScript) + Tailwind in `web/`.
- Tests: vitest. API tests use @cloudflare/vitest-pool-workers.
- Local dev: `wrangler dev` in `api/` (init local DB with `npm run db:init`), `vite dev` in `web/`.
- Timezone rules (load-bearing, see ARCHITECTURE.md): date-only slots are plain `YYYY-MM-DD` strings, never shifted; timed slots store `HH:MM` plus the originating IANA timezone; never store naive local times as UTC.
- Theming: event `theme` JSON → CSS custom properties. Adding a theme field must be a localized change (type/default + one UI control + CSS var reference).
- Auth: global bearer secret for host API access; per-event `admin_token` for host UI; invitees are unauthenticated.
