# Session Notes — 2025-12-24

## Scope
- Target: OWASP Juice Shop (local)
- Base URL: `http://127.0.0.1:3000` (also reachable via `http://localhost:3000`)
- Account used: admin@juice-sh.op (local lab)
- Capture methods: Chrome DevTools (Network / Application)

## Authentication artifacts observed

### Login response (token issuance)
- Endpoint: `POST /rest/user/login`
- Observation: response body includes `authentication.token` (JWT-like)
- No `Set-Cookie` observed on the login API response (in this flow)

Evidence
- Raw HTTP: `docs/evidence/http/20251224-01-login-api-response.txt` (token redacted)
- Screenshots:
  - `docs/evidence/screenshots/20251224-s03-login-request-headers.png`
  - `docs/evidence/screenshots/20251224-s04-login-response-headers.png`
  - `docs/evidence/screenshots/20251224-s05-login-response-body.png`
  - `docs/evidence/screenshots/20251224-s06-login-no-set-cookie.png`

### Post-login browser cookies
- Cookies present after login (DevTools view):
  - `token` (JWT-like format)
  - `language`
  - `cookieconsent_status`
  - `welcomebanner_status`

Cookie attribute quick check (DevTools → Application → Cookies)
- `token`: appears JavaScript-accessible (not HttpOnly); Secure not indicated; SameSite not explicitly shown in this session
- Others: no defensive attributes indicated in this session (no HttpOnly / Secure / SameSite shown)

Evidence
- `docs/evidence/screenshots/20251224-s02-cookies-after-login.png`

### Representative authenticated request behavior
- Observation: authenticated requests include `Authorization: Bearer <token>`
- A `token=<...>` cookie is also present in the request headers during capture
- Note: exact token values were compared during inspection; sensitive values are redacted in exports

Evidence
- `docs/evidence/screenshots/20251224-s07-basket-add-request-headers.png`
- `docs/evidence/screenshots/20251224-s09-basket-add-response-headers.png`
- `docs/evidence/screenshots/20251224-s10-basket-add-response-body.png`

## Notes / follow-ups
- Check whether any unsafe endpoint accepts cookie-only authentication (would change CSRF relevance).
- Run controlled checks against one state-changing endpoint:
  - cookie reliance (remove Cookie)
  - auth source (remove Authorization)
  - Origin behavior (remove / spoof Origin) in replay tests
  - XSRF header behavior (change/remove `X-XSRF-TOKEN` if present)
- Findings writeups:
  - `docs/findings/20251224-f01-token-storage-cookie.md`
  - `docs/findings/20251224-f02-token-cookie-flags.md`
  - `docs/findings/20251225-f03-csrf-basketitems.md`
