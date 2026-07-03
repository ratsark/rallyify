# Rallyify Architecture

## Stack

| Layer | Choice | Rationale |
|---|---|---|
| API | Cloudflare Workers (Hono, TypeScript) | Serverless, edge-distributed, no idle cost |
| Database | Cloudflare D1 (SQLite) | Serverless SQLite, same ecosystem, no separate process |
| Frontend | Cloudflare Pages (React + Vite, TypeScript) | SPA, CDN-distributed, integrates cleanly with Workers |
| Styling | Tailwind CSS + CSS custom properties | Per-event theming via injected CSS variables |

Two deployments on a single Cloudflare account:
- `api.rallyify.com` → Workers (Hono API + D1)
- `rallyify.com` → Pages (React/Vite SPA)

---

## API Routes

All routes served by the Workers deployment. Auth is described below.

### Public (no auth)
| Method + Path | Purpose |
|---|---|
| `GET /events/:id` | Get event + slots (public fields only) |
| `POST /events/:id/responses` | Submit or update a response |

### Host (API token required)
| Method + Path | Purpose |
|---|---|
| `POST /events` | Create event |
| `GET /events` | List all events |
| `PATCH /events/:id` | Update event (title, description, deadline, theme, status) |
| `DELETE /events/:id` | Delete event |
| `POST /events/:id/slots` | Add slot |
| `PATCH /events/:id/slots/:slotId` | Edit slot |
| `DELETE /events/:id/slots/:slotId` | Delete slot |
| `GET /events/:id/responses` | Get all responses |
| `POST /events/:id/finalize` | Finalize a slot |

The API is the primary interface for Claude-driven event management. The host web UI calls the same endpoints.

---

## Frontend Pages

Served by the Pages deployment. All data fetched from the Workers API.

### Public
| Route | Purpose |
|---|---|
| `/` | Landing page |
| `/events/:id` | Responder page — view slots, submit availability |
| `/events/:id/done` | Confirmation after responding |

### Host (admin token in URL → cookie)
| Route | Purpose |
|---|---|
| `/host` | Dashboard — list of events |
| `/host/events/new` | Create event |
| `/host/events/:id` | Event detail — response grid, finalize, edit slots/theme |

---

## Database Schema

```sql
CREATE TABLE events (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  description TEXT,
  deadline TEXT,                    -- ISO 8601, nullable
  status TEXT NOT NULL DEFAULT 'open',  -- open | finalized
  finalized_slot_id TEXT,
  allow_invitee_slots INTEGER NOT NULL DEFAULT 0,
  theme TEXT NOT NULL DEFAULT '{}', -- JSON
  invite_token TEXT NOT NULL UNIQUE,
  admin_token TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE slots (
  id TEXT PRIMARY KEY,
  event_id TEXT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  date TEXT NOT NULL,               -- "YYYY-MM-DD", never timezone-shifted
  start_time TEXT,                  -- "HH:MM" in originating timezone, nullable
  end_time TEXT,
  timezone TEXT,                    -- IANA tz of host at creation time, nullable
  created_at TEXT NOT NULL
);

CREATE TABLE responses (
  id TEXT PRIMARY KEY,
  event_id TEXT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  email TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE slot_responses (
  id TEXT PRIMARY KEY,
  response_id TEXT NOT NULL REFERENCES responses(id) ON DELETE CASCADE,
  slot_id TEXT NOT NULL REFERENCES slots(id) ON DELETE CASCADE,
  availability TEXT NOT NULL,       -- yes | no | maybe
  UNIQUE(response_id, slot_id)
);
```

### Timezone strategy
- `date` is a plain `YYYY-MM-DD` string — date-only slots are never timezone-shifted.
- `start_time` / `end_time` are stored in the host's originating timezone (`timezone` column).
- The responder page detects the viewer's local timezone and renders converted times, showing the originating timezone for reference.
- Never store naive local times as UTC — this is the one thing painful to retrofit.

---

## Theming

Each event's `theme` JSON field contains:

```json
{
  "accent": "#6366f1",
  "background": "#ffffff",
  "textColor": "#111827",
  "buttonColor": "#6366f1",
  "headerImageUrl": "",
  "welcomeMessage": "",
  "font": "inter"
}
```

The responder page injects these as CSS custom properties on `<html>`:

```html
<html style="
  --color-accent: #6366f1;
  --color-bg: #ffffff;
  --color-text: #111827;
  --color-button: #6366f1;
  --font-family: 'Inter', sans-serif;
">
```

**Extensibility rule:** adding a new theme field requires only: (1) add the key to the theme type/default, (2) add a control to the host UI / API docs, (3) reference the new CSS variable in the template. No structural changes.

Font options (MVP): `inter`, `playfair` (serif, elegant), `mono` (technical).

---

## Auth

| Actor | Mechanism |
|---|---|
| Claude / scripts | `Authorization: Bearer $RALLYIFY_API_SECRET` on all host routes |
| Host web UI | Per-event `admin_token` in URL on first visit sets a cookie; subsequent requests validate cookie |
| Invitees | No auth — identified by name on response submission |

`RALLYIFY_API_SECRET` is a Cloudflare Workers secret (`wrangler secret put`), never in the repo. The host web UI uses per-event `admin_token` values rather than the global secret, so individual event admin links can be bookmarked safely.

---

## Directory Structure

```
rallyify/
├── api/                            # Cloudflare Workers (Hono)
│   ├── src/
│   │   ├── index.ts                # Hono app entry, route registration
│   │   ├── routes/
│   │   │   ├── events.ts           # CRUD + list
│   │   │   ├── slots.ts            # Slot management
│   │   │   ├── responses.ts        # Submit/get responses
│   │   │   └── finalize.ts         # Finalize slot
│   │   ├── middleware/
│   │   │   └── auth.ts             # Bearer token validation
│   │   ├── lib/
│   │   │   └── theme.ts            # Theme type + defaults
│   │   └── schema.sql
│   ├── wrangler.toml
│   └── package.json
│
└── web/                            # Cloudflare Pages (React/Vite)
    ├── src/
    │   ├── main.tsx
    │   ├── App.tsx                  # Router
    │   ├── pages/
    │   │   ├── Landing.tsx
    │   │   ├── EventRespond.tsx     # Public responder page
    │   │   ├── EventDone.tsx
    │   │   ├── HostDashboard.tsx
    │   │   ├── HostEventNew.tsx
    │   │   └── HostEventDetail.tsx  # Response grid + finalize + theme editor
    │   ├── components/
    │   │   ├── EventTheme.tsx       # Injects CSS vars from theme JSON
    │   │   ├── SlotGrid.tsx         # Response grid (host view)
    │   │   ├── SlotPicker.tsx       # Availability picker (invitee view)
    │   │   └── ThemeEditor.tsx      # Theme form fields
    │   └── lib/
    │       ├── api.ts               # Typed API client
    │       └── theme.ts             # Theme type + defaults (shared with api/)
    ├── index.html
    ├── vite.config.ts
    └── package.json
```

---

## Deployment

```toml
# api/wrangler.toml
name = "rallyify-api"
main = "src/index.ts"
compatibility_date = "2025-04-01"
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "rallyify-db"
database_id = "<to be set>"

[[routes]]
pattern = "api.rallyify.com"
custom_domain = true
```

`RALLYIFY_API_SECRET` set via `wrangler secret put RALLYIFY_API_SECRET`.

Pages deployment configured in the Cloudflare dashboard, pointing to `web/` with build command `vite build`.
