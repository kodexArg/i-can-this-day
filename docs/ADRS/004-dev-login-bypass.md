# ADR-004 — Dev-only Login Bypass: /dev/auth/ form (double-gated)

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/ADRS/001-stack.md · docs/ADRS/003-auth-flow.md

## Context

ADR-003 routes all real logins through Cognito + Google, a round-trip of ~3 seconds
that requires internet and is blocked until the infra prerequisites (Google IdP,
ClientSecret on `hoy-puedo-client`) land. Local development and Playwright smoke tests
need a fast, in-process way to obtain a logged-in session without touching Cognito.
SROA solved this with a `dev_auth` Django app; we mirror it.

## Decision

- **Module**: a separate Django app at `backend/dev_auth/`. Self-contained; no edits
  to the main app's auth code.
- **Exposed URL**: `/dev/auth/login/` — a minimal HTML form with one `email` field.
  POST creates the `User` if missing, logs them in via Django's session machinery,
  and redirects to `/`.
- **User creation behavior**: any submitted email yields a working session.
  `User.username` = email, `User.email` = email, `first_name`/`last_name` derived from
  the email local-part (e.g. `marcos@x.com` → `Marcos`). No `SocialAccount` row is
  created — intentional divergence from prod; the bypass is for the SESSION only.
- **Logout**: standard `/accounts/logout/` (allauth) — no separate dev logout needed.
- **Double gate (BOTH must be true to mount the URL)**:
  - `settings.DEBUG is True`
  - `os.environ.get("DEV_AUTH_ENABLED") == "1"`
  - Contract: root `urls.py` conditionally includes `dev_auth.urls`. If either gate
    is missing the URL is not registered — visitors get a plain 404, not a redirect
    or 403.
- **Production safety**: `dev_auth` stays in `INSTALLED_APPS` (migrations stay in
  mainline), but its URL conf is unmounted in prod. The view itself ALSO refuses to
  execute if the gates fail, as defense-in-depth.
- **Tests**: pytest may (a) hit `/dev/auth/login/` via the test client — exercises the
  real flow (Playwright smoke tests use this), or (b) use `client.force_login()` for
  unit-scale tests — fast, skips the dev-auth code path entirely.
- **CI environment**: `DEV_AUTH_ENABLED` is NEVER set in any prod task definition or
  CI deploy job. It IS set in `.env.example` and in the Playwright runner's environment.

## Guardrails / Threat Model

- Two-gate design: a single misconfiguration cannot expose the URL. Forgetting
  `DEBUG=False` in prod still leaves `DEV_AUTH_ENABLED=1` missing (and vice versa).
- A startup-time Django `system check` SHOULD log a loud warning whenever the
  `dev_auth` URL conf is mounted, so a misconfigured prod container surfaces it
  immediately in logs.
- Code review checklist: any PR touching `dev_auth/urls.py`, `dev_auth/views.py`, or
  the conditional include in root `urls.py` requires explicit review. Enforced
  socially in v1; CODEOWNERS later.
- The bypass creates Users NOT linked to Cognito. If a developer later signs in via
  real Cognito with the same email, allauth will either link or create a duplicate —
  explicitly acceptable for v1; dev DBs are disposable.
- The form has NO CSRF exception, NO password, and is unstyled. Intentionally not
  "polished" — anyone seeing it in prod will recognize it as misconfigured tooling.

## Consequences

- Dev loop: ~50 ms to log in vs ~3 s for real Cognito. Significant iteration win.
- Test suite is decoupled from infra availability — tests pass whether or not Cognito
  is provisioned.
- The codebase carries ~80 lines that exist solely to be excluded from production.
  Acceptable for v1; revisit if the cost grows.
- The dev-auth session does NOT exercise allauth's `user_signed_up` signal path. Tests
  for signal-based logic (e.g. auto-populate display name from Cognito profile) must
  use a different mechanism.
- The Cognito provisioning blockers (infra ADR-A-009) no longer block development —
  only the eventual real-user smoke test on `hoy-puedo.dev.grupoalvs.com`.

## Alternatives Considered

- **Management command only (`manage.py devlogin`)** — zero HTTP surface but requires
  cookie copy/paste. Rejected: ergonomic cost too high for daily iteration.
- **Custom auth backend + `?devuser=` query-param** — smallest code surface but
  query-string credentials end up in logs; devs may accidentally paste the URL into
  bug reports. Rejected: leakage risk worse than dual-gate.
- **No bypass; use real Cognito with localhost callback + VCR fixtures for tests** —
  most production-faithful but ~3 s per login, blocked on infra, and breaks offline
  development. Rejected: blocks today's work and degrades iteration speed permanently.

## Open Follow-ups

- Add a Django startup-time `system check` that warns if `dev_auth` URLs are mounted.
- When INFRA.md is written, list `DEV_AUTH_ENABLED` in the "MUST NOT be set in prod"
  checklist alongside `DEBUG`.
- Consider a CI lint that fails if a deployed task definition contains
  `DEV_AUTH_ENABLED`.
