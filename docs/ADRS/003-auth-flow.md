# ADR-003 — Authentication: django-allauth + Cognito (Google IdP)

- Status: Accepted (blocked on infra prerequisites — see §Prerequisites)
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md §5 · docs/ADRS/001-stack.md · docs/ADRS/002-hosting-topology.md
- Mirrors: AWS infra ADR-A-008, ADR-A-009 (from docs-new-infra-grupoalvs-com)

## Context

PRD §5 requires Google-only login — no email/password, no other IdP. ADR-001 selected
Django + django-allauth as the app framework and auth library. ADR-002 fixed the
canonical hostname at `hoy-puedo.dev.grupoalvs.com`. This ADR decides the OAuth chain
shape, who holds the user directory, and what the canonical callback URLs are. SROA
solved this same problem and we mirror its decisions to stay aligned with the org SSO
pattern.

## Decision

- **Library**: `django-allauth` with the built-in `amazon-cognito` provider. No
  additional OAuth library.
- **Flow shape**: app → Cognito `/authorize?identity_provider=Google` → Google consent
  → Cognito callback → app's allauth callback. The Cognito Hosted UI is NEVER rendered
  to the user — the `identity_provider=Google` query param bypasses it entirely.
- **Callback URL (canonical)**: `https://hoy-puedo.dev.grupoalvs.com/accounts/amazon-cognito/login/callback/`.
  Must be registered on the Cognito App Client.
- **Login entry URL**: `/accounts/amazon-cognito/login/` (allauth default).
- **Logout URL**: `/accounts/logout/` (allauth default). Clears Django session only —
  does NOT call Cognito's global sign-out endpoint in v1.
- **Session backend**: Django's default cookie session (server-side store via the
  `django_session` table). Cookie flags: `HttpOnly=True`, `Secure=True`, `SameSite=Lax`.
- **Identity key**: `SocialAccount.uid` populated from the Cognito `sub` claim. NEVER
  key off email — protects against Google Workspace email recycling.
- **What we store on first login**: Django `User` row (username = Cognito sub, email =
  Google email, first_name + last_name from Google profile). `SocialAccount` row links
  the user to the Cognito identity. Avatar URL is stored on the social-account
  `extra_data` JSON and re-read on each session refresh — no local copy.
- **Tokens on the client**: NONE. No JWT, no access token, no refresh token in cookies
  or local storage. The Django session cookie is the entire session state.
- **What requires login** (mirrors PRD §5): tapping a candidate day to open the day
  modal, creating a new event. Viewing the calendar and participation counts does NOT
  require login.
- **Anonymous tap-then-login UX**: when an anonymous user taps a candidate day, allauth
  handles the round-trip and a `?next=` deep-link returns the user to the same event URL
  after auth. The day modal opens automatically after redirect — NOT a second tap.
- **Session TTL**: Django default (2 weeks of inactivity). Re-evaluated only if user
  research shows pain.

## Prerequisites (BLOCKERS — infra side, not ours)

- **Cognito App Client `hoy-puedo-client`** must be created on user pool
  `us-east-1_PL2hwbYXD` with `GenerateSecret=true`. The allauth Auth Code flow requires
  the client secret. SROA's `sroa-client` lacks a secret — infra ADR-A-009 documents
  that mistake. We do not repeat it.
- **Callback URL** `https://hoy-puedo.dev.grupoalvs.com/accounts/amazon-cognito/login/callback/`
  registered on the client. A localhost variant for dev is optional but recommended.
- **Allowed OAuth scopes** on the client: `openid email profile`. No
  `aws.cognito.signin.user.admin`.
- **`UserPoolIdentityProvider` resource for Google** on the user pool. As of 2026-05-23
  this resource does NOT exist — the secret `alvs/shared/google-oauth` is populated but
  the AWS-side IdP resource is unprovisioned (infra ADR-A-009). Auth will not work until
  this lands.
- **Secrets Manager entry** `alvs/dev/hoy-puedo/cognito` with keys `COGNITO_DOMAIN`,
  `COGNITO_CLIENT_ID`, `COGNITO_CLIENT_SECRET`, `COGNITO_USER_POOL_ID`. Injected into
  the Fargate task via `valueFrom`.
- **allauth `SocialApp` row** seeded at deploy time (data migration or one-shot
  management command) so the provider knows the client ID and secret. SROA does this on
  first deploy.
- **Cognito User Pool Groups**: not used for authorization (infra ADR-A-008 — Cognito =
  authn only, Django = authz).

## Consequences

- Same library version and code paths as SROA — lower risk and shared mental model for
  whoever maintains both apps.
- The flow is fully server-side; no client-side OAuth state to debug.
- The app cannot ship a working login until the infra prerequisites land. Everything
  else (calendar, participation toggling, event creation) can be built and tested behind
  a dev-only login bypass — to be defined in ADR-004.
- Cognito is the central identity broker; if a second app joins the org, users share
  `sub`s. App-level data stays isolated because it keys off `User.id`, not Cognito sub.
- We do NOT call Cognito's global sign-out on logout → a user who logs out and logs in
  again may re-bind their Google identity without a consent prompt. Acceptable for v1.
- Adding email/password or a second IdP later requires switching from the forced
  `identity_provider=Google` param to the Cognito Hosted UI picker — a single config
  change, no architectural shift.

## Alternatives Considered

- **allauth + Google directly (no Cognito)** — ships today, no infra blocker, but
  diverges from org SSO pattern. Rejected: org alignment outweighs the short-term unblock.
- **allauth + Cognito Hosted UI (user picks IdP)** — adds an extra screen contradicting
  PRD §12.4 (WhatsApp-first minimalism). Rejected: PRD says Google-only for v1.
- **Custom OIDC via authlib** — re-solves a problem allauth already solved; higher
  maintenance forever. Rejected.

## Open Follow-ups

- ADR-004 will define the dev-only login bypass module (mirror SROA's `dev_auth/`
  pattern, env-gated, strictly out of production).
- INFRA.md (later doc) will track the Cognito client + Google IdP provisioning checklist
  as deploy prerequisites.
- Revisit Cognito global sign-out if "stuck logged in" complaints appear.
