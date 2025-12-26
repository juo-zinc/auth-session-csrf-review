# Auth / Session / CSRF Review Checklist (Juice Shop local)

Last updated: 2025-12-26

## Scope
- Target: OWASP Juice Shop (local)
- Base URL: http://localhost:3000 (also reachable via http://127.0.0.1:3000)
- Focus areas:
  - Auth token acquisition & usage (JWT / bearer)
  - Token storage & cookies
  - Cookie security flags
  - CSRF-related behavior on state-changing endpoints
  - Origin/CORS observations relevant to CSRF assumptions (behavioral, non-exploit)

## Environment & tools
- Browser: Chrome / Edge (DevTools used for cookie inspection)
- Interception / replay: Burp Suite (Repeater)
- Account: admin@juice-sh.op (local lab)

---

## A. Auth token flow (login → authenticated requests)

### A1. Confirm login returns a token
- Checked: `/rest/user/login` response body includes `authentication.token` (JWT).
- Evidence:
  - `docs/evidence/http/20251224-01-login-api-response.txt`
  - `docs/evidence/screenshots/20251224-s05-login-response-body.png`

### A2. Confirm subsequent API calls use Authorization: Bearer
- Checked: authenticated requests include `Authorization: Bearer <JWT>` after login.
- Evidence:
  - `docs/evidence/screenshots/20251224-s03-login-request-headers.png`
  - `docs/evidence/screenshots/20251224-s07-basket-add-request-headers.png`

### A3. Confirm login does not set a server session cookie
- Checked: login response headers contain no `Set-Cookie` (no server session cookie issuance observed in this flow).
- Evidence:
  - `docs/evidence/screenshots/20251224-s04-login-response-headers.png`
  - `docs/evidence/screenshots/20251224-s06-login-no-set-cookie.png`
  - `docs/evidence/http/20251224-01-login-api-response.txt`

---

## B. Token storage (browser-side) & cookie inventory

### B1. Identify where the token is stored (observed)
- Checked: a `token` cookie is visible in browser storage after login (observed in this lab session).
- Evidence:
  - `docs/evidence/screenshots/20251224-s02-cookies-after-login.png`

### B2. Record observed cookies after login
- Observed cookies (example):
  - `token`
  - `language`
  - `cookieconsent_status`
  - `welcomebanner_status`
  - `XSRF-TOKEN` (present in this lab session)
- Evidence:
  - `docs/evidence/screenshots/20251224-s02-cookies-after-login.png`

---

## C. Cookie security flags (HttpOnly / Secure / SameSite)

### C1. Check cookie flags for token and other app cookies
- Checked flags for:
  - `token` cookie
  - `language`, `cookieconsent_status`, `welcomebanner_status` (and any other cookies visible)
- Evidence:
  - `docs/evidence/screenshots/20251224-s11-token-cookie-flags.png`

### C2. Findings mapping
- F-01: Token stored in cookie (JavaScript-accessible)
- F-02: Token cookie missing security flags (HttpOnly / Secure / SameSite)
- References:
  - `docs/findings/20251224-f01-token-storage-cookie.md`
  - `docs/findings/20251224-f02-token-cookie-flags.md`

---

## D. CSRF checks (state-changing endpoint: /api/BasketItems)

Endpoint tested:
- `POST /api/BasketItems/` (path observed with trailing slash in this lab; treat as the BasketItems create endpoint)

Threat-model note (important):
- In the observed flow, the endpoint requires `Authorization: Bearer <JWT>`.
- Browsers do not automatically attach `Authorization` headers in cross-site requests the way they attach cookies.
- Therefore, results below are recorded as “behavioral checks” for XSRF/Origin handling, not as a proof of a practical CSRF exploit.

### D1. Baseline (authenticated request succeeds)
- Expectation: returns 200 OK with created basket item data.
- Evidence:
  - `docs/evidence/screenshots/20251225_basketitems_baseline_request.png`
  - `docs/evidence/screenshots/20251225_basketitems_baseline_response.png`

### D2. No Authorization header (should be rejected)
- Checked: request without Authorization fails.
- Evidence:
  - `docs/evidence/screenshots/20251225_basketitems_baseline_no_auth_request.png`
  - `docs/evidence/screenshots/20251225_basketitems_no_auth_response.png`

### D3. No Cookie header (control)
- Checked: request without cookies still succeeds as long as Authorization is present.
- Evidence:
  - `docs/evidence/screenshots/20251225_basketitems_no_cookie_request.png`
  - `docs/evidence/screenshots/20251225_basketitems_no_cookie_response.png`

### D4. Origin header behavior (no observable Origin validation in this test)
- Checked (non-browser client replay):
  - No `Origin` header: request succeeds.
  - Modified `Origin: https://evil.example`: request still succeeds.
- Note: Origin checks are primarily meaningful for browser requests; replay tools can omit/spoof Origin. This section records observed behavior only.
- Evidence:
  - `docs/evidence/screenshots/20251225_basketitems_no_origin_request.png`
  - `docs/evidence/screenshots/20251225_basketitems_no_origin_response.png`
  - `docs/evidence/screenshots/20251225_basketitems_origin_evil_request.png`
  - `docs/evidence/screenshots/20251225_basketitems_origin_evil_response.png`

### D5. XSRF header behavior (no enforcement observed under Bearer-token auth)
Tested on the same endpoint with Authorization present:
- Case 1: `X-XSRF-TOKEN: AAA` → 200 OK
- Case 2: `X-XSRF-TOKEN: BBB` → 200 OK
- Case 3: `X-XSRF-TOKEN` missing → 200 OK
- Interpretation: the `X-XSRF-TOKEN` header value did not affect request acceptance in this observed flow.
- Evidence:
  - `docs/evidence/screenshots/20251225-s14-basketitems-csrf-xsrf-aaa-request.png`
  - `docs/evidence/screenshots/20251225-s15-basketitems-csrf-xsrf-aaa-response.png`
  - `docs/evidence/screenshots/20251225-s16-basketitems-csrf-xsrf-bbb-request.png`
  - `docs/evidence/screenshots/20251225-s17-basketitems-csrf-xsrf-bbb-response.png`
  - `docs/evidence/screenshots/20251225-s18-basketitems-csrf-xsrf-missing-request.png`
  - `docs/evidence/screenshots/20251225-s19-basketitems-csrf-xsrf-missing-response.png`

### D6. Finding mapping
- F-03: `/api/BasketItems` does not enforce XSRF token validation in the observed bearer-token flow (and Origin behavior did not change acceptance during replay tests)
- Reference:
  - `docs/findings/20251225-f03-csrf-basketitems.md`

---

## E. Report & documentation mapping
- Final report:
  - `docs/REPORT.md`
- Session notes:
  - `docs/notes/20251224-session-notes.md`

---

## Pre-publish sanity checks
- [ ] Redact JWTs / sensitive tokens in screenshots and text exports
- [ ] Ensure filenames referenced above exist in repo (no dead links)
- [ ] Keep evidence paths stable (do not rename after report is written)
- [ ] Keep findings IDs consistent across files (F-01 / F-02 / F-03)
