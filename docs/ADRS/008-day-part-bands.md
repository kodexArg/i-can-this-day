# ADR-008 — Day-Part Time Bands: 06/13/20 cultural ranges, shown in modal

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md §3 (Day-part), §11 open question #1, §13 (UI copy), §11 out-of-scope #5 (single TZ) · docs/ADRS/006-data-model.md

## Context

PRD §3 names the three day-parts (Mañana / Tarde / Noche) but explicitly defers exact hour ranges to an ADR (PRD §11 open question #1). PRD §11 out-of-scope item #5 commits us to a single server-configured time zone. ADR-006 stored `day_part` as a `TextChoices` enum without any time semantics. We now pin the hour ranges and how/where they surface in the UI.

## Decision

**Time zone**
- Server-wide `TIME_ZONE` setting: `America/Argentina/Buenos_Aires` (UTC-3, no DST observed since 2009).
- All "today / past / future" reasoning uses this zone. No per-user zone in v1 (PRD §11 out-of-scope #5).
- The setting is configurable in Django settings but has exactly one value in v1. Changing it later is a coordinated operation (would re-classify which days are "past" at the moment of cutover).

**Day-part hour ranges (local time)**
- **Mañana**: 06:00 – 13:00
- **Tarde**: 13:00 – 20:00
- **Noche**: 20:00 – 06:00 (wraps to the following calendar day)
- The 13:00 and 20:00 boundaries belong to the LATER day-part (inclusive start, exclusive end). At 13:00:00 exactly, the current day-part is Tarde, not Mañana.
- Coverage is exhaustive — every minute of every day belongs to exactly one day-part.

**Where the ranges ARE shown**
- Day modal: under each day-part heading, a small secondary-text subtitle renders the range as `06:00 – 13:00` (and equivalents).
- Subtitle styling: smaller font, muted color. Visual hierarchy = heading first, range second.

**Where the ranges are NOT shown**
- Calendar cells on the event view (only counts appear there, per PRD §4.2).
- The create-event form (organizer marks dates, not day-parts).
- The landing page event list.
- Any URL or page title.

**How the ranges are USED in logic (server-side)**
- "Is THIS day-part on TODAY already past?" — used to disable currently-past day-part toggles within today's date. Example: if local time is 14:30 today, today's Mañana toggle is rendered non-interactive (greyed) in the modal; Tarde and Noche remain interactive.
- Calendar cells for fully-past days (`CandidateDay.date < today`) follow PRD §4.1: greyed-out, non-clickable. Hour ranges do not factor into past-day rendering — only into today's partial-day greying.
- Past-event cutoff (PRD §11 open question #10): an event is "past" when its latest `CandidateDay.date`'s Noche window ends at 06:00 the morning after. Practical effect: the last day stays interactive through the wrap-around Noche window.

**Display copy (canonical Spanish, mirrors PRD §13)**
- Headings: `Mañana`, `Tarde`, `Noche` (verbatim from PRD §13).
- Range format: `HH:MM – HH:MM` with the en-dash (`–`, not the hyphen `-`).
- Noche subtitle: `20:00 – 06:00` (the "next day" is not annotated in the UI — users intuit the wrap).

## Consequences

- First-time users immediately understand what each label means without trial-and-error voting.
- Today's already-past day-parts disable themselves naturally, preventing "I marked Mañana at 4pm" confusion.
- The Noche wrap-around (20:00 → 06:00 next day) means a candidate day's Noche window extends into the early hours of the following calendar date. Noche is past only at 06:01 the next day, not at 23:59 on the day itself.
- The single-TZ commitment keeps all calendar logic in server time — no `Intl.DateTimeFormat` work on the client. Compatible with the server-rendered HTMX architecture from ADR-001.
- If the product ever serves users outside `America/Argentina/Buenos_Aires`, the bands stay culturally correct but the TZ assumption breaks. Cross-TZ support would require a new ADR.
- Three small subtitles per modal (one per day-part heading). Negligible visual cost.

## Alternatives Considered

- **Wide cultural bands, HIDDEN (internal-only)** — cleaner UI but first-time users have no anchor for what "Tarde" means; edge cases like "why is Mañana greyed at 14:00?" surprise users. Rejected.
- **Narrow event-hour bands, shown (e.g., 09–13 / 14–20 / 20–00)** — introduces gaps in coverage where no day-part is selectable; excludes breakfast and late-night events. Rejected: imposes opinion the PRD never asked for.
- **No bands at all (pure cosmetic labels)** — simplest model but loses the "is today's day-part past" disable behavior and gives first-time users zero hint of meaning. Rejected.

## Open Follow-ups

- If smoke tests surface confusion at the wrap-around boundary, consider annotating Noche subtitle as `20:00 – 06:00 (día siguiente)`. Defer until a real complaint surfaces.
- The `America/Argentina/Buenos_Aires` assumption should be reflected in the dev `.env.example` and in the Fargate task definition env vars when ADR-005's deploy pipeline lands.
- DST is not currently a concern (Argentina has not observed DST since 2009). If the country re-adopts DST, band boundaries do not change — the IANA zone name handles it automatically without needing a fixed UTC offset.
- PRD §11 open question #10 (past-event cutoff) is resolved here — remove from the PRD's open-questions list in a future PRD revision.
