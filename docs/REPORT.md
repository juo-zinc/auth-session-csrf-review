# Auth / Session / CSRF Review Report (OWASP Juice Shop – Local)

**Project:** auth-session-csrf-review  
**Target:** OWASP Juice Shop (local instance)  
**Base URL:** http://localhost:3000 (also tested via http://127.0.0.1:3000)  
**Test window:** 2025-12-24 to 2025-12-25  
**Account used:** admin@juice-sh.op (local lab)

---

## 1. Executive summary

A short, evidence-driven review was performed focusing on authentication token handling, cookie attributes, and CSRF-related controls for a representative state-changing endpoint.

Key observations:
- The application returns a JWT in the login response body and subsequent API calls use `Authorization: Bearer <token>`.
- After login, a `token` cookie is visible in the browser. The login response does not set cookies via `Set-Cookie`, suggesting client-side persistence of the token (e.g., JavaScript writing the token into a cookie).
- For `/api/BasketItems`, requests succeed even when `X-XSRF-TOKEN` is modified or omitted. In the observed flow, the endpoint still requires a valid `Authorization` header, which reduces the likelihood of classical browser-based CSRF for this endpoint under the tested conditions.

Overall risk posture (in this observed flow):
- Primary risk concentrates on token exposure and client-side token storage.
- XSRF enforcement appears ineffective for the tested endpoint, but exploitability is constrained by the requirement for an explicit bearer token.

---

## 2. Scope and approach

### In scope
- Auth/session behavior observable from browser and Burp traffic
- Client-side token persistence and cookie security attributes
- CSRF/XSRF-related behavior for `/api/BasketItems` (state-changing)

### Out of scope
- Exploit development, destructive actions, DoS/stress testing
- Full application vulnerability coverage beyond the endpoints tested
- Server-side source code review

### Tools / methods
- Browser DevTools (Application/Cookies, Network)
- Burp Suite (Repeater) for controlled request variants
- Evidence captured under `docs/evidence/`

---

## 3. Findings overview

| ID   | Title | Severity | Status |
|------|-------|----------|--------|
| F-01 | JWT returned in body; token persisted client-side as a `token` cookie (Bearer token used for API auth) | Medium | Documented |
| F-02 | `token` cookie lacks common defensive attributes (not HttpOnly; Secure/SameSite not explicitly shown in this session) | Medium | Documented |
| F-03 | `/api/BasketItems` accepts state-changing requests without effective XSRF enforcement (as tested under Bearer-token auth) | Low | Documented |

---

## 4. Detailed findings

### F-01 — JWT returned in body; token persisted client-side as a `token` cookie (Medium)

**Description**  
After a successful login, the response body includes a JWT token and subsequent API calls use `Authorization: Bearer <token>`. In the browser, a `token` cookie is present after login. The login response does not set cookies via `Set-Cookie`, suggesting that token persistence happens on the client side (e.g., JavaScript storing the token into a cookie).  
Representative authenticated requests include `Authorization: Bearer <token>` while a `token` cookie is also present in request headers (dual presence).

**Impact**  
Client-side persistence of authentication tokens increases exposure in the presence of client-side injection issues (e.g., XSS). If tokens are long-lived, the impact of token theft increases. Dual presence (Bearer + cookie) can also increase the chance of inconsistent enforcement if any endpoint later begins accepting cookie-based authentication.

**Evidence**
- Login API response body (token redacted):  
  `docs/evidence/http/20251224-01-login-api-response.txt`
- Browser cookies after login:  
  `docs/evidence/screenshots/20251224-s02-cookies-after-login.png`
- Representative authenticated request headers (Bearer + cookie):  
  `docs/evidence/screenshots/20251224-s07-basket-add-request-headers.png`

**Note**  
Cookie attribute hardening is covered in F-02.

---

### F-02 — `token` cookie lacks common defensive attributes (Medium)

**Description**  
In Chrome DevTools, the `token` cookie appears JavaScript-accessible (i.e., not HttpOnly) and does not indicate `Secure`. `SameSite` is not explicitly shown/set in this session.

**Impact**  
- Without `HttpOnly`, the token can be read by JavaScript running in the origin, increasing the impact of any XSS.
- Without an explicit `SameSite` policy, cookie behavior may be less constrained in cross-site contexts (CSRF relevance depends on whether any endpoint accepts cookie-based authentication).
- `Secure` is primarily relevant when the application is served over HTTPS. In a local HTTP lab environment this may be expected; however, in production it should be enforced to reduce exposure on insecure transport.

**Evidence**
- Cookie flags view:  
  `docs/evidence/screenshots/20251224-s11-token-cookie-flags.png`

---

### F-03 — `/api/BasketItems` accepts state-changing requests without effective XSRF enforcement (Low)

**Description**  
Request variants were sent to `/api/BasketItems` to evaluate CSRF/XSRF control behavior. The endpoint accepts requests even when the `X-XSRF-TOKEN` header value is altered or omitted.

Observed behavior:
- Baseline state-changing request succeeds (200 OK).
- `X-XSRF-TOKEN` set to an arbitrary value still succeeds (200 OK).
- `X-XSRF-TOKEN` omitted still succeeds (200 OK).
- `Authorization` omitted results in 401 Unauthorized.
- Cookies omitted while keeping `Authorization` still succeeds (cookies not required to authorize this action).
- During replay tests (non-browser client), removing/spoofing `Origin` did not change acceptance.

**Impact**  
The tested endpoint does not appear to enforce an effective anti-CSRF token check. Under the observed flow, a valid bearer token in the `Authorization` header is required, and browsers do not automatically attach such headers in cross-site requests. This reduces the likelihood of classical CSRF against this endpoint in the tested scenario.  
However, ineffective XSRF enforcement still represents a control gap and can become higher risk if the authorization model changes (e.g., cookie-based auth for state-changing actions, token auto-injection by the browser, or compromised frontend context).

**Evidence (XSRF token variants)**
- XSRF header set to `AAA` → 200 OK:  
  `docs/evidence/screenshots/20251225-s14-basketitems-csrf-xsrf-aaa-request.png`  
  `docs/evidence/screenshots/20251225-s15-basketitems-csrf-xsrf-aaa-response.png`
- XSRF header set to `BBB` → 200 OK:  
  `docs/evidence/screenshots/20251225-s16-basketitems-csrf-xsrf-bbb-request.png`  
  `docs/evidence/screenshots/20251225-s17-basketitems-csrf-xsrf-bbb-response.png`
- XSRF header missing → 200 OK:  
  `docs/evidence/screenshots/20251225-s18-basketitems-csrf-xsrf-missing-request.png`  
  `docs/evidence/screenshots/20251225-s19-basketitems-csrf-xsrf-missing-response.png`

**Evidence (authorization/cookie/origin checks)**
- No `Authorization` header → 401 Unauthorized:  
  `docs/evidence/screenshots/20251225_basketitems_baseline_no_auth_request.png`  
  `docs/evidence/screenshots/20251225_basketitems_no_auth_response.png`
- No cookies (with `Authorization`) → 200 OK:  
  `docs/evidence/screenshots/20251225_basketitems_no_cookie_request.png`  
  `docs/evidence/screenshots/20251225_basketitems_no_cookie_response.png`
- `Origin` spoofed (replay observation) → 200 OK:  
  `docs/evidence/screenshots/20251225_basketitems_origin_evil_request.png`  
  `docs/evidence/screenshots/20251225_basketitems_origin_evil_response.png`

---

## 5. Recommendations (summary)

- Avoid persisting bearer tokens in JavaScript-accessible storage (including non-HttpOnly cookies). Prefer a single, consistent authentication channel and strong token lifecycle controls (short TTL, rotation/refresh patterns).
- If cookies are used for authentication in any flow, apply defensive attributes (`HttpOnly`, `Secure`, explicit `SameSite`) and ensure unsafe methods have CSRF protections. Note: if the `token` cookie is created client-side (no server `Set-Cookie`), `HttpOnly` cannot be applied; remediation requires removing the cookie or switching to a server-issued session cookie.
- If an anti-CSRF mechanism is intended (e.g., XSRF cookie/header pattern), enforce it consistently on state-changing endpoints and reject requests when missing/incorrect.
- Review origin-handling and CORS behavior for API endpoints to ensure it matches the intended threat model.

---

## 6. Limitations

This assessment is a short, non-destructive review performed on a local training instance and focuses on a small set of observed endpoints and flows. It should not be treated as comprehensive application security testing.

---

## 7. Appendix: repository mapping

- Findings:
  - `docs/findings/20251224-f01-token-storage-cookie.md`
  - `docs/findings/20251224-f02-token-cookie-flags.md`
  - `docs/findings/20251225-f03-csrf-basketitems.md`
- Evidence:
  - `docs/evidence/http/`
  - `docs/evidence/screenshots/`
  - `docs/evidence/burp/` (if exported later)
