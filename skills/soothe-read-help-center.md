---
name: soothe-read-help-center
description: >-
  Read Soothe's customer, provider and partner help-centre content from the mirror Soothe
  serves at help.soothe.com, using its manifest as the index. Use this to answer
  questions about Soothe bookings, cancellations, refunds, SoothePass and provider
  policy. Do not use it to book, cancel or modify an appointment — Soothe publishes no
  API for that.
api: openapi/soothe-help-center-mirror-openapi.json
operations:
  - health_health_get
  - manifest_manifest_json_get
  - serve_page__full_path__get
---

# Read the Soothe help centre

Soothe has no public developer program. The one contract it serves is the OpenAPI 3.1.0
schema of the FastAPI mirror behind `https://help.soothe.com`, and that surface is
anonymous, read-only content retrieval. Everything below is grounded in an operationId
that exists verbatim in `openapi/soothe-help-center-mirror-openapi.json`.

## Authentication

None. No `securityScheme` is declared and none is required. Send no credentials.

## Steps

1. **Check freshness first.** Call `health_health_get` — `GET https://help.soothe.com/health`.
   It returns `{"status":"ok","pages":<n>,"mirrored_at":"<ISO8601>", ...}`. The
   `mirrored_at` timestamp is the age of the content: this is a point-in-time snapshot,
   not live help-centre content. Observed 2026-08-28: mirrored 2026-06-22, 541 pages.
   Always report that timestamp alongside any answer you draw from this source.

2. **Get the index.** Call `manifest_manifest_json_get` —
   `GET https://help.soothe.com/manifest.json`. It returns a `pages[]` array of every
   mirrored path. There is no search endpoint and no pagination: match the user's
   question against these paths yourself. Useful paths include
   `/docs/booking-an-appointment`, `/docs/cancellation-policy`,
   `/docs/requesting-a-refund`, `/docs/soothepass`, `/docs/managing-your-appointment`,
   `/docs/trust-and-safety`.

3. **Fetch the page.** Call `serve_page__full_path__get` —
   `GET https://help.soothe.com/{full_path}` with a path taken from the manifest.

## Errors

- **404** — `{"detail":"Page not found"}`. The path is not in the mirror. Re-read the
  manifest rather than guessing another path; the mirror does not redirect or suggest.
  This response is returned live but is *not* declared in the specification.
- **422** — `{"detail":[{"loc":[...],"msg":"...","type":"..."}]}` (`HTTPValidationError`).
  The path parameter failed validation.
- The envelope is FastAPI's `detail` field on `application/json`, **not** RFC 9457
  `application/problem+json`. See `errors/soothe-problem-types.yml`.

## Rules and limits

- **Do not call `proxy_api_api__full_path__options` on `/api/{full_path}`.** All three
  methods on that path share one operationId, so the specification cannot distinguish
  them, and the operation is an undocumented passthrough to an upstream service.
- **No rate limits are documented** and no `RateLimit-*`, `X-RateLimit-*` or
  `Retry-After` header is returned. Rate yourself conservatively; you will get no
  runtime back-off signal. See `rate-limits/soothe-rate-limits.yml`.
- **This surface is read-only.** There is no create, update, cancel or refund operation.
  If the user wants to book, change or cancel an appointment, or request a refund, tell
  them to use the Soothe app or `https://www.soothe.com/` — do not attempt
  `https://api.soothe.com`, which returns a sign-in form for every request.
- **Content may be stale.** Because the mirror is a snapshot, verify anything
  time-sensitive — pricing, promotions, service-area coverage — against
  `https://www.soothe.com/` before relying on it.
