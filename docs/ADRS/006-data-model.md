# ADR-006 — Data Model: Event / CandidateDay / Participation (normalized)

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md §3 (glossary) · docs/ADRS/001-stack.md

---

## 1. Context

PRD §3 names the domain entities (Event, Candidate day, Day-part, Participation,
Organizer, Participant) but does not pin a relational schema. ADR-001 fixed Django
and Postgres as the stack. This ADR locks the relational shape that all subsequent
code, tests, and admin views depend on, resolving ambiguities in PRD §11 where
applicable.

---

## 2. Decision

### `Event` — the scheduling proposal as a whole

- `id` — auto bigint PK
- `code` — short opaque string, UNIQUE (generation strategy deferred to ADR-007)
- `name` — string, max 120 chars, REQUIRED (default "Evento sin nombre" if creation
  form policy later marks it optional per PRD §4.4)
- `organizer` — FK → `auth.User`, ON DELETE PROTECT (no silent event loss if a user
  account is ever removed; hard block is intentional in v1)
- `created_at`, `updated_at` — timestamps

### `CandidateDay` — a date the organizer marked as eligible

- `id` — auto bigint PK
- `event` — FK → `Event`, ON DELETE CASCADE
- `date` — date (local calendar date, no time component)
- UNIQUE (`event_id`, `date`) — a date cannot be a candidate twice in the same event
- `created_at` — timestamp (rows are insert/delete only, never updated)

### `Participation` — one user's claim they can attend a specific cell

- `id` — auto bigint PK
- `user` — FK → `auth.User`, ON DELETE CASCADE
- `candidate_day` — FK → `CandidateDay`, ON DELETE CASCADE
- `day_part` — CharField(choices=DayPart.choices) where `DayPart` is a `TextChoices`
  with members `MORNING='morning'`, `AFTERNOON='afternoon'`, `NIGHT='night'`; string
  values are the storage form; labels (Mañana / Tarde / Noche) come from the UI copy
  table (PRD §13)
- UNIQUE (`user_id`, `candidate_day_id`, `day_part`) — a user cannot double-vote the
  same cell
- `created_at`, `updated_at` — timestamps

### Implicit relationships (derived, not stored)

- **Organizer**: `Event.organizer_id`. No separate `organizer` table.
- **Participant**: any `User` with ≥1 `Participation` row joined to a `CandidateDay`
  of that Event. Derived via query; not a stored relation.
- **Past vs future event**: derived from `MAX(CandidateDay.date)` per event compared
  to today in the server's configured local zone. No stored `is_past` flag — calculated
  at read time so it never drifts.

### What is NOT modeled (intentional non-goals for v1)

- No `Invitation` table — possession-of-link is the access token (PRD §6).
- No `EventLifecycle` / status enum — no formal lifecycle in v1 (PRD §3.3).
- No `Notification`, `Comment`, or `Attachment` tables (PRD §11 out-of-scope).
- No soft-delete (`deleted_at`) columns anywhere — hard deletes; FK cascades handle
  cleanup.
- No per-user display-name override — name + avatar are re-read from `User` /
  `SocialAccount.extra_data` per ADR-003.

---

## 3. Consequences

- The UNIQUE constraint on `Participation(user, candidate_day, day_part)` makes the
  "toggle then save" UI naturally idempotent at the DB level: a save view can
  `Participation.objects.filter(user=u, candidate_day=cd).delete()` + recreate the
  chosen day-parts atomically without worrying about duplicates.
- Calendar render is a single ORM call: filter by `candidate_day__event__code`,
  annotate by date and day_part with `Count`. No raw SQL needed.
- Removing a `CandidateDay` cascades to remove all its participations. The PRD open
  question (§11) is now answered: cascade-delete, no soft-delete recovery.
- Adding a new day-part later (e.g. "late night") = add a `TextChoices` member. No
  schema migration beyond the new choices entry; constraint columns unchanged.
- `Event.code` is the only public identifier in the URL pattern `/<event-code>/`.
  Lookups MUST go through `code`, never `id`. The `id` field stays internal-only.
- `created_at`/`updated_at` everywhere relevant gives Django admin and any future
  audit log a free timeline at zero cost.

---

## 4. Alternatives Considered

- **Denormalized counts on CandidateDay** (signals maintain `morning_count` etc.) —
  faster calendar render but premature at this scale; risk of drift. Rejected.
- **Two-table with JSON `candidate_days` on Event** — fewer tables but loses FK from
  `Participation` to `CandidateDay`, forcing validation in app code. Rejected.
- **Bitmask compact** (one `Participation` row per user×day with a 3-bit slots mask)
  — compact but unidiomatic ORM and schema-fragile if a 4th day-part is added.
  Rejected.

---

## 5. Open Follow-ups

- ADR-007: event-code generation algorithm (alphabet, length, collision-retry
  strategy).
- PRD §11 "Concurrent save collisions" — resolved: the toggle-then-save view wraps
  delete+create in a `select_for_update` transaction (implementation detail, no
  further ADR needed).
- PRD §11 "Past-event cutoff definition" — resolved here: an event is "past" when
  `MAX(CandidateDay.date) < today` in the server's local zone. Day-part end times
  are display-only.
- PRD §11 "Account identity stability" — resolved by ADR-003 (keyed off Cognito
  `sub`; a Google email change doesn't fork the identity). No model-level
  intervention needed.
- Future: a periodic `RECONCILE` management command if denormalized counts are ever
  added.
