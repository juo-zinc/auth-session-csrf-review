# F-03: Bearer token is the gating control for `POST /api/BasketItems/`; XSRF header appears unused in this flow

## Summary
For `POST /api/BasketItems/`, request acceptance does not change when `X-XSRF-TOKEN` is set to arbitrary values or omitted. In replay (non-browser) tests, the server response remained `200 OK` when `Origin` was omitted or set to an arbitrary value, suggesting no observable server-side Origin gating for this endpoint in the tested flow.

Based on the observed behavior, this workflow is protected primarily by Bearer-token authentication. Under a classic browser CSRF threat model, cross-site requests do not automatically include custom `Authorization` headers, so CSRF is not practical for this endpoint in the tested flow.

This finding is recorded as a hardening / future-risk note: if any state-changing endpoint becomes cookie-authenticated (intentionally or accidentally), the currently observed lack of XSRF enforcement and lack of observable Origin gating would become relevant to CSRF exposure.

## Environment
- Target: OWASP Juice Shop (local)
- Base URL: `http://localhost:3000`
- Endpoint: `POST /api/BasketItems/`
- Tooling: Burp Suite Repeater (replay / variant tests)

## Test matrix (evidence-driven)

### Case 1 — Baseline (authorized request succeeds)
- Change: none
- Observed: `200 OK`

Evidence
- `docs/evidence/screenshots/20251225_basketitems_baseline_request.png`
- `docs/evidence/screenshots/20251225_basketitems_baseline_response.png`

### Case 2 — Auth source check (remove Cookie)
- Purpose: verify whether cookies participate in authorization for this endpoint
- Change: remove `Cookie` header entirely (keep `Authorization`)
- Observed: `200 OK` (cookies not required in this flow)

Evidence
- `docs/evidence/screenshots/20251225_basketitems_no_cookie_request.png`
- `docs/evidence/screenshots/20251225_basketitems_no_cookie_response.png`

### Case 3 — Auth source check (remove Authorization)
- Purpose: verify whether `Authorization: Bearer` is required for authorization
- Change: remove `Authorization: Bearer ...` (keep Cookie)
- Observed: `401 Unauthorized`

Evidence
- `docs/evidence/screenshots/20251225_basketitems_baseline_no_auth_request.png`
- `docs/evidence/screenshots/20251225_basketitems_no_auth_response.png`

### Case 4 — Origin handling (replay test: remove / spoof Origin)
- Note: this is a replay observation (non-browser). Origin headers can be omitted/spoofed outside the browser context.
- Change A: remove `Origin` → Observed `200 OK`
- Change B: set `Origin: https://evil.example` → Observed `200 OK`

Evidence
- `docs/evidence/screenshots/20251225_basketitems_no_origin_request.png`
- `docs/evidence/screenshots/20251225_basketitems_no_origin_response.png`
- `docs/evidence/screenshots/20251225_basketitems_origin_evil_request.png`
- `docs/evidence/screenshots/20251225_basketitems_origin_evil_response.png`

### Case 5 — XSRF header handling (change/remove `X-XSRF-TOKEN`)
- Change A: `X-XSRF-TOKEN: AAA` → Observed `200 OK`
- Change B: `X-XSRF-TOKEN: BBB` → Observed `200 OK`
- Change C: remove `X-XSRF-TOKEN` → Observed `200 OK`

Evidence
- `docs/evidence/screenshots/20251225-s14-basketitems-csrf-xsrf-aaa-request.png`
- `docs/evidence/screenshots/20251225-s15-basketitems-csrf-xsrf-aaa-response.png`
- `docs/evidence/screenshots/20251225-s16-basketitems-csrf-xsrf-bbb-request.png`
- `docs/evidence/screenshots/20251225-s17-basketitems-csrf-xsrf-bbb-response.png`
- `docs/evidence/screenshots/20251225-s18-basketitems-csrf-xsrf-missing-request.png`
- `docs/evidence/screenshots/20251225-s19-basketitems-csrf-xsrf-missing-response.png`

Interpretation: In the tested Bearer-token flow, `X-XSRF-TOKEN` does not appear to be validated or required for request acceptance.

## Evidence boundary / test hygiene

- Note: Because POST /api/BasketItems/ is state-changing, repeating the exact same captured request can trigger application/data-layer errors (e.g., duplicate/unique-constraint violations) and produce HTTP 500. This behavior is treated as a replay artifact and is not interpreted as XSRF enforcement (which would typically manifest as a clean authorization/CSRF rejection such as 401/403).
- To keep the XSRF-variant test meaningful, each variant request should be sent against a clean state (e.g., reset basket state) or use a non-duplicate payload (e.g., different ProductId) so that the result reflects request acceptance controls rather than application uniqueness rules.

## Impact assessment
- Practical CSRF (as-tested): unlikely for this endpoint under the observed design, because a valid `Authorization: Bearer` token is required and is not automatically attached by browsers in cross-site requests.
- Hardening / future risk:
  - If any unsafe endpoint becomes cookie-authenticated, missing XSRF enforcement + lack of observable Origin gating may increase CSRF exposure.
  - Independent of CSRF: if tokens are exposed (e.g., via client-side token storage weaknesses or XSS), an attacker can reuse the Bearer token to perform authenticated actions.

## Severity (suggested)
Informational / Low (as-tested), due to Bearer-token requirement and lack of a practical CSRF path in this flow.

## Recommendations
- Keep state-changing APIs protected via non-cookie credentials (`Authorization` header) and avoid introducing cookie-only authentication paths for unsafe methods.
- If an XSRF mechanism is intended for this workflow, enforce it as a server-side gating control:
  - reject requests with missing/invalid XSRF token for unsafe methods
  - consider Origin/Referer checks as defense-in-depth for sensitive state-changing operations
