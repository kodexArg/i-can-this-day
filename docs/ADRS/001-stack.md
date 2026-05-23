# ADR-001 — Stack: Django 6 + HTMX (server-rendered)

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md

## Context

The PRD describes a lightweight daily event-tracking web app with a small,
focused feature set: users log in, record their day, and see their history.
A stack decision is required before any implementation begins to avoid
mid-build pivots. A sibling project (SROA) already uses an Astro + Django +
Channels split; this app must not replicate that architecture — the two-service
split and WebSocket layer are overkill here. However, Django patterns that SROA
validated, especially around Cognito auth, are worth reusing.

## Decision

- Runtime: Python 3.13
- Framework: Django 6
- UI: server-rendered Django templates + HTMX (`django-htmx`) for partial swaps
- Client JS budget: `htmx.min.js` only (~14 KB); no SPA bundle, no React/Vue
- Auth: django-allauth with the `amazon-cognito` provider (mirrors SROA's solved flow)
- DB: PostgreSQL via Django ORM
- Async / WebSockets: NOT used — no Django Channels, no Daphne; plain WSGI
  (gunicorn) is sufficient; cell refreshes are HTMX swaps, not realtime push
- Package manager: `uv` (per global Python-first rule)
- Container base: official `python:3.13-slim`

## Consequences

- **Hiring pool**: standard Django developers with a small HTMX learning curve;
  HTMX is HTML attributes, not a separate paradigm.
- **Deploy shape**: single Fargate service — no frontend/backend split like SROA.
  ALB host rule routes `hoy-puedo.dev.grupoalvs.com` to one target group.
- **Dev loop**: `manage.py runserver` + browser; no frontend build step, no
  `npm install`; hot-reload is Django's native autoreload.
- **Auth**: reuses the django-allauth flow from AWS infra ADR-A-009, including
  the pending `ClientSecret` provisioning step and the Google IdP blocker until
  the AWS infra side ships it.
- **Admin**: Django admin provides a free internal data viewer for
  events/participations — support tooling at zero extra cost.
- **Locked out**: real-time WebSocket push (adding Channels later is possible
  but a re-architecture); React/Vue islands (would require a JS build pipeline).

## Alternatives Considered

- **FastAPI + HTMX + Jinja2** — smaller container but no django-allauth
  equivalent for Cognito; hand-rolling the OIDC flow re-solves a solved problem.
- **Next.js (App Router) on Node Fargate** — largest hiring pool but 5–10x more
  client JS, heavier deploy, and abandons Python-first preference; weight
  mismatch with the product's simplicity.
- **SvelteKit on Node Fargate** — leanest JS option but smaller community and
  no first-class Cognito library; net cost higher than the Django path.
- **SROA stack verbatim (Astro + Django + Channels)** — explicitly out of scope
  per the "different stack than SROA" directive; two-service split is overkill.

## Open Follow-ups

- ADR-002 will fix hosting topology: Fargate task sizing, ALB rule, RDS
  database name.
- ADR-003 will fix auth flow specifics: callback URL pattern, allauth settings,
  Cognito client provisioning checklist.
