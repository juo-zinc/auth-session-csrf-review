# F-02: token cookie hardening not observed (HttpOnly/Secure/SameSite)

## Summary
After login, a cookie named `token` is present. In Chrome DevTools (Application → Cookies), the cookie appears to be JavaScript-accessible (i.e., not HttpOnly) and does not indicate `Secure`. `SameSite` is not explicitly shown/set in this session.

If the `token` cookie is relied on for authentication by any endpoint (now or in the future), missing defensive attributes weaken protection against token theft (XSS) and can increase cross-site request risk depending on how cookies are sent.

## Environment
- Target: OWASP Juice Shop (local)
- Base URL: `http://127.0.0.1:3000` / `http://localhost:3000`
- Evidence source: Chrome DevTools → Application → Cookies

## Observation (evidence-based)
- Cookie name: `token`
- Observed properties (DevTools view):
  - HttpOnly: not indicated (cookie appears JavaScript-accessible)
  - Secure: not indicated
  - SameSite: not explicitly shown/set in this session

Evidence
- `docs/evidence/screenshots/20251224-s02-cookies-after-login.png`
- (If available) `docs/evidence/screenshots/20251224-s11-token-cookie-flags.png`

## Impact
- Token theft / XSS amplification:
  - If an XSS occurs, a JavaScript-accessible auth token can be read and exfiltrated, enabling session takeover via token replay.
- Transport hardening (deployment-dependent):
  - If the application is served over HTTPS in non-local environments, missing `Secure` would allow the cookie to be sent over non-HTTPS channels (where applicable).
- Cross-site request risk (conditional):
  - If `SameSite` is not explicitly set (or is set to permissive values), cross-site cookie sending can become more permissive.
  - This only becomes a practical CSRF concern if any state-changing endpoint accepts cookie-based authentication.

## Severity (suggested)
Medium (context-dependent).
Primary driver: token exposure to JavaScript (XSS impact).
Secondary driver: whether cookie-based authentication is accepted by any endpoint.

## Recommendations
- Prefer keeping bearer tokens out of client-readable cookies:
  - If the cookie is not required, remove it to avoid accidental reliance.
  - If a cookie must be used for auth, strongly consider a server-issued session cookie rather than a client-readable token.
- If the `token` cookie is required:
  - Set `HttpOnly; Secure; SameSite=Lax` (or `Strict` where feasible)
  - Keep tokens short-lived and rotate/expire aggressively
  - Ensure unsafe methods have explicit CSRF protections if cookies are used for authentication
- If the app uses Bearer tokens via `Authorization` already, remove duplicate `token` cookies to avoid accidental future reliance and reduce client-side exposure.
