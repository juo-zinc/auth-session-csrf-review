# auth-session-csrf-review

Evidence-driven review of authentication/session behavior and CSRF-related controls based on a local OWASP Juice Shop instance.

> Non-destructive testing only. No secrets (JWTs/cookies/password hashes) are committed. Sensitive values in screenshots/requests are redacted.

## Target

- Local Juice Shop: `http://127.0.0.1:3000` (also reachable via `http://localhost:3000`)

## Deliverables (start here)

- **Final report:** `docs/REPORT.md`
- **Checklist:** `docs/CHECKLIST.md`
- **Findings (per-issue writeups):** `docs/findings/`
- **Evidence (screenshots/raw captures):** `docs/evidence/`

## Scope (what this repo covers)

- Observable auth/session behavior (token issuance, storage, request usage)
- Cookie attributes relevant to token exposure
- CSRF/XSRF-related behavior for a representative state-changing endpoint (`POST /api/BasketItems/`)

Out of scope: exploit development, destructive testing, DoS/stress testing, and source code review.

## What this repo contains

This repo is organized around **evidence → findings → final report**.

- `docs/REPORT.md` — consolidated report (main deliverable)
- `docs/findings/` — individual finding writeups
  - `20251224-f01-token-storage-cookie.md`
  - `20251224-f02-token-cookie-flags.md`
  - `20251225-f03-csrf-basketitems.md`
- `docs/evidence/` — raw evidence supporting the findings
  - `docs/evidence/screenshots/` — screenshots (DevTools/Burp/requests/responses)
  - `docs/evidence/http/` — raw HTTP exports / notes (if any)
  - `docs/evidence/burp/` — Burp exports (if any)
- `docs/notes/` — working notes / observations used during investigation
- `docs/CHECKLIST.md` — lightweight checklist used to keep the review consistent

## Findings index

- **F-01** JWT returned on login; token persisted client-side as a cookie (dual presence with Bearer auth)  
  `docs/findings/20251224-f01-token-storage-cookie.md`

- **F-02** `token` cookie lacks common defensive attributes (observed in this session)  
  `docs/findings/20251224-f02-token-cookie-flags.md`

- **F-03** XSRF/CSRF-related observations on `POST /api/BasketItems/`  
  (as-tested: requests require `Authorization` and return 401 without it, limiting classic CSRF in this flow)  
  `docs/findings/20251225-f03-csrf-basketitems.md`

## Evidence naming convention

Screenshots follow a sortable format:

```text
YYYYMMDD-sNN-<area>-<topic>-<details>-(request|response).png
