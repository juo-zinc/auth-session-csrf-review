# F-01: JWT returned on login and persisted client-side as a JavaScript-accessible `token` cookie (dual presence with Authorization)

## Summary
After login, the application returns an authentication token in the login API response body (`authentication.token`). A `token` cookie is present in the browser after login, and representative authenticated requests use `Authorization: Bearer <token>`. The presence of the token both as a bearer credential and as a browser cookie increases exposure to client-side compromise (notably XSS) and can lead to inconsistent enforcement if any endpoint later relies on the cookie as an authentication channel.

Cookie defensive attributes (HttpOnly/Secure/SameSite) are discussed in F-02.

## Environment
- Target: OWASP Juice Shop (local)
- Base URL: `http://127.0.0.1:3000` / `http://localhost:3000`
- User: admin (local test)
- Evidence sources: Chrome DevTools (Network + Application), Burp traffic captures (replay/inspection)

## Evidence

### 1) Login response contains token in JSON
- Request: `POST /rest/user/login`
- Response: `200 OK`
- Token field: `authentication.token`

Files
- `docs/evidence/http/20251224-01-login-api-response.txt` (token redacted)
- `docs/evidence/screenshots/20251224-s05-login-response-body.png`

### 2) No server-issued session cookie observed on login response
- Observation: login response headers do not include `Set-Cookie` for an auth/session cookie in this flow.

Files
- `docs/evidence/screenshots/20251224-s04-login-response-headers.png`
- `docs/evidence/screenshots/20251224-s06-login-no-set-cookie.png`

### 3) `token` cookie present after login (browser-side persistence)
- Observation: DevTools → Application → Cookies shows a `token` cookie present after login.

Files
- `docs/evidence/screenshots/20251224-s02-cookies-after-login.png`

### 4) Authenticated requests use Bearer token; token cookie is also present in request headers
- Observation: a representative state-changing request includes:
  - `Authorization: Bearer <JWT-like token>`
  - `Cookie: ... token=<value> ...`
- Note: token values are treated as sensitive and are redacted in exported artifacts; matching was verified during inspection.

Files
- `docs/evidence/screenshots/20251224-s07-basket-add-request-headers.png`
- `docs/evidence/screenshots/20251224-s09-basket-add-response-headers.png`
- `docs/evidence/screenshots/20251224-s10-basket-add-response-body.png`

## Impact
- Primary (token theft / XSS amplification):
  - If any XSS exists, a JavaScript-accessible auth token cookie can be read and exfiltrated, enabling token replay and account compromise until expiry/rotation.
- Secondary (design hardening / consistency):
  - Token present as both `Authorization` and cookie increases the chance of inconsistent enforcement **if** any endpoint begins accepting the cookie as an authentication channel (intentionally or accidentally).
- CSRF note (context):
  - For `/api/BasketItems` as tested, requests are rejected without `Authorization` (401), which limits practical classic CSRF in this flow. See F-03.

## Severity (suggested)
Medium (context-dependent)
- Becomes High if XSS is present, if tokens are long-lived, or if any sensitive endpoint accepts cookie-only auth without robust CSRF controls.

## Recommendations
### Preferred (single-channel bearer-token design)
- Keep `Authorization: Bearer <token>` as the only authentication channel for API calls.
- Avoid storing the same auth token in browser cookies (especially non-HttpOnly cookies).
- Use short-lived access tokens; if persistence is required, use refresh-token rotation and strict expiry.

### If a cookie must be used for auth
- Use a server-issued session cookie rather than a client-readable token where feasible.
- Apply defensive attributes (see F-02):
  - `HttpOnly; Secure; SameSite=Lax` (or `Strict` where compatible)
- Scope with least privilege (`Path`, avoid wide `Domain`, short TTL).
- Ensure unsafe methods have CSRF defenses when cookies are used for authentication (token validation and/or Origin/Referer enforcement as defense in depth).

## Reproduction (non-intrusive)
1) Login and capture `POST /rest/user/login` response; confirm `authentication.token` exists.
2) Check DevTools → Application → Cookies; confirm `token` cookie exists after login.
3) Trigger an authenticated action and confirm `Authorization: Bearer ...` is present and `Cookie: ... token=...` is also present in captured traffic.
