# Rallyify Requirements

**Meta-goal:** make organizing events low-friction and even enjoyable. Scheduling negotiation is the pain point; every requirement below should be judged by whether it *reduces back-and-forth* or *removes a decision from the host's plate*.

**Core model (shared vocabulary):**
- **Host** — creates the event, owns the decision. Currently a single personal user (the developer); no multi-tenant host accounts in MVP.
- **Invitee** — responds with availability.
- **Event** — a thing being scheduled: title, description, proposed time slots, and a visual theme.
- **Time slot** — a candidate date, optionally with a time range.
- **Response** — an invitee's availability across all slots.

---

## 1. MVP

The MVP must support the full loop: create → propose slots → invite → collect availability → host decides. It also exposes a first-class API/CLI interface for the host (Claude) to manage events, and supports per-event visual theming.

### Event creation — host web UI
1. **Host creates an event with a title (required) and description (optional).**
2. **Host adds N time slots, each a date with an optional time range (start–end).** Slots can be date-only ("Sat Mar 14") or timed ("Sat Mar 14, 7–9 pm").
3. **Host can add, edit, and remove slots before and after invites go out.**
4. **Host sets a response deadline (optional).**
5. **Host enables or disables invitee-proposed slots per event (default off).** *(Data model shipped in MVP; full re-notification flow in Phase 2.)*

### Event creation — Claude/API interface
6. **A REST API (or equivalent) supports all event-management operations:** create event, add/edit/remove slots, set theme, view responses, finalize. This is the primary interface for programmatic or Claude-driven event creation.
7. **API access is restricted to the host** (simple auth token is sufficient for MVP).

### Per-event visual theming
8. **Each event has a configurable theme** stored as structured data on the event. MVP fields: color accent, background color, text color, button color, header image URL, welcome message, font (from a small curated set).
9. **Theme fields are set via the API or host UI.** Adding new theme fields in the future should require only a localized code change (schema field + UI control + template reference) — no architectural refactoring.
10. **The responder page renders the event's theme** via CSS custom properties injected at the page level.

### Invitation
11. **Host shares a unique link per event — no account required for invitees to respond.**
12. **Invitee identifies themselves by name (required) and email (optional) when responding.**

### Responses
13. **Invitee marks each slot as Yes / No / Maybe.**
14. **Invitee can edit their response until the deadline or until the host finalizes.**

### Results & decision
15. **Host sees a response grid: who said what for each slot, with per-slot counts** ("5 yes, 1 maybe, 2 no").
16. **Best slot(s) are visually highlighted** (most Yes, fewest No).
17. **Host finalizes a slot; the event page updates to show the confirmed time to all invitees.**

### Cross-cutting MVP concerns
18. **All slots display in the viewer's local timezone, shown explicitly.** *(Flagged as deceptively complex — see below.)*
19. **Responder pages are mobile-first / responsive.**

---

## 2. Phase 2

1. **Email notifications:** invite sent, response received (batched), event finalized.
2. **Automated reminders to non-responders** before the deadline.
3. **"Add to calendar" link/download** for the finalized event (ICS + Google/Outlook links).
4. **Invitee-proposed time slots** (host-configurable toggle, default off): invitees can suggest additional slots; host approves before they go live; existing invitees are notified to respond to new slots.
5. **Calendar availability import for invitees** (Google/Outlook OAuth to auto-flag conflicts). *(Flagged: OAuth + privacy complexity.)*
6. **Optional host accounts** to manage multiple events and reuse invitee groups.
7. **Change notifications** when the host edits an event after invites are sent.

---

## 3. Phase 3+

1. **Smart slot suggestions** derived from invitee calendars or response patterns.
2. **Auto-finalize rules** ("pick the first slot where everyone says Yes"; "decide at deadline").
3. **Conflict-resolution nudges** — surface near-misses ("only Dana can't do Thu — swap to Fri and everyone's in").
4. **Ranked/preference voting** beyond Yes/Maybe/No.
5. **Recurring-event scheduling** ("the slot that works most weeks").
6. **Fun / social layer:** slot reactions/emoji, "who's excited" signal, celebratory finalize moment.
7. **Natural-language event creation** via Claude ("dinner with the team next week, evenings").
8. **Group memory / templates** ("the book club usually does Sunday afternoons").
9. **Real-time collaborative response view.** *(Flagged: periodic refresh gets 80% of value for 10% of cost — do that first.)*

---

## Deceptively-complex requirements (plan accordingly)

- **Timezone handling (MVP #18).** Store slots in unambiguous absolute form (UTC + originating timezone); render per-viewer. Never store naive local times. Edge cases: date-only slots (must *not* be timezone-shifted), DST transitions. Get the data model right up front — retrofitting is expensive.
- **Invitee-proposed slots (Phase 2 #4).** Existing responses become incomplete when a slot is added, requiring re-notification and potentially re-responses. Also needs abuse controls (per-invitee limits, host removal).
- **Calendar integration (Phase 2 #5).** Per-provider OAuth, token refresh, quotas, and privacy: scope to free/busy only, not event details.
- **Real-time updates (Phase 3 #9).** Periodic refresh (every 30s) achieves most of the benefit at a fraction of the complexity of WebSockets or SSE.

---

## Design principles

- **No account required to respond** — the single biggest friction reducer. Inviolable.
- **The finalize step is core, not polish** — it's what distinguishes Rallyify from a raw poll.
- **Theming extensibility over completeness** — ship a small set of theme fields; make adding new ones trivial.
- **Claude is a first-class host interface** — the API should be as capable as the web UI.
