# ADR-007 — Event-Code Generation: 11-char [A-Z0-9], retry on collision

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md §3 (Event code), §11 (open question on code length/alphabet) · docs/ADRS/006-data-model.md

## Context

PRD §3 defines `Event code` as a short opaque alphanumeric string embedded in the URL (`/<event-code>/`) and shows two examples (`HOI35L145KK`, `BAS529G45K3`) — both 11-char uppercase `[A-Z0-9]`. PRD §11 explicitly deferred the exact alphabet, length, and collision strategy to the ADR phase. ADR-006 fixed the schema with `Event.code` as a UNIQUE string column. This ADR pins the generation rules.

## Decision

- **Alphabet**: `[A-Z0-9]` — 36 characters. Uppercase only; never lowercase. No symbols. No URL-encoded characters.
- **Length**: exactly 11 characters. Fixed-width. Never padded, never trimmed.
- **Keyspace**: 36^11 ≈ 1.3 × 10^17. At one million events the birthday-paradox collision probability is ~10^-5; we will not hit it in v1.
- **Generator**: Python `secrets.choice()` over the alphabet, 11 times. NEVER `random.random()` / `random.choice()` (CSPRNG vs PRNG). One call to the generator = one candidate code.
- **Persistence path**: an `Event` row is built with the generated `code` and `.save()` is attempted inside an atomic transaction. The UNIQUE constraint on `Event.code` (per ADR-006) is the authoritative collision check.
- **Collision-retry policy**: on `IntegrityError` from the UNIQUE constraint, regenerate the code and retry. **Maximum 3 attempts total** (1 original + 2 retries). After 3 failures raise a `RuntimeError`; this is treated as a system fault (500 + log + alert), not a user-facing error — at our keyspace it should never occur in practice and signals a deeper issue (CSPRNG broken, alphabet corrupted, or DB-side bug).
- **Validation at URL routing**: the URL pattern for the event view is `^/(?P<code>[A-Z0-9]{11})/$`. Codes that do not match never reach the view — they 404 at the routing layer. No lowercase fallback / case-insensitive lookup.
- **Validation at model save**: a model-level `RegexValidator(r'^[A-Z0-9]{11}$')` on `Event.code` ensures any non-generator code path (admin form, data import, test fixture) is also rejected at the validation layer.
- **Code lifetime**: codes are stable forever. No rotation, no expiry, no re-issuance. A code is generated once at creation and never changes (PRD NFR-5 — URL stability).
- **Lookup**: all `Event` lookups in views and tests MUST go through `code` (e.g., `get_object_or_404(Event, code=...)`). The primary key `id` is INTERNAL — it never appears in URLs, templates, or responses. Reinforces possession-of-link as the access token (PRD §6).

## Consequences

- The PRD's possession-of-link security model holds: with a 1.3 × 10^17 keyspace an attacker cannot meaningfully enumerate codes. Brute-force at 1000 requests/sec would take 10^7 years to cover 0.01% of the space. Rate-limiting at the ALB is still good practice but not load-bearing for confidentiality.
- Forever-stable codes mean any URL ever shared in a WhatsApp thread continues to work for the life of the event. Aligns with PRD NFR-5 and NFR-6.
- 11 chars in a URL like `/HOI35L145KK/` is on the longer side for hand-typed entry, but the product flow is copy-paste from a chat thread; hand-typing is not a use case.
- Visual ambiguity between `0`/`O` and `1`/`I` is unimportant because codes are never dictated verbally (link is shared as a clickable URL via WhatsApp/Instagram). If user research later shows hand-typing is happening, switch to Crockford base32 — a one-line alphabet change + migration, explicitly out of scope for v1.
- Using a CSPRNG (`secrets`) means a security review can sign off without auditing the RNG path. `random` would have failed that review.
- The 3-retry cap is high enough that we will never hit it in practice and low enough that a CSPRNG bug surfaces fast instead of looping forever.

## Alternatives Considered

- **Crockford base32, 10 chars (no I/L/O/U)** — better spoken-aloud readability but diverges from PRD example shape and codes aren't dictated verbally anyway. Rejected.
- **Nanoid, 8 chars [A-Z0-9]** — shorter URLs but new dependency for a 10-line problem, and 36^8 keyspace is 50000× smaller. Rejected.
- **Sqids / hashids over `Event.id`** — shortest possible but reversible and enumerable — breaks possession-of-link security model entirely. Rejected.

## Open Follow-ups

- Add ALB/Django rate-limiting on event-creation endpoints to prevent abuse (separate ADR or just a setting; out of scope here).
- If a future internationalization layer ever wants emoji codes or non-ASCII alphabets, this ADR is the place to revisit — not a code change in passing.
- Operational runbook entry: if the "code generation exhausted 3 retries" error ever fires, page on it — almost certainly a system fault, not natural collision.
