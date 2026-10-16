---
doc: prd
status: approved
---

# Privacy Copilot for AI Workflows — Product Requirements

A privacy-checkpoint tool for developers, analysts, and legal consultants who want to detect and redact sensitive information before submitting text to an external AI service.
Source: `scope.md > The Unique Kernel`, `scope.md > Who It's For`.

---

## The Core Journey

Source: `scope.md > The Core Loop`, `scope.md > What "Working" Looks Like`.

### Public path (no account required)

1. User opens the web interface. They see a single-page tool: a text area, a "Scan" button, and a short plain-English note explaining that text may be sent to Gemini for analysis and that detection is not guaranteed.
2. User pastes text (up to 2,000 characters) and presses "Scan."
3. The frontend sends `POST /api/v1/scans/public/`.
4. The backend runs deterministic regex detection on the full text locally.
5. The backend sends the minimum necessary context to Gemini for contextual analysis and risk explanation.
6. The backend validates the model's structured response against a strict schema.
7. The backend merges local and model findings, runs deterministic redaction, and returns a structured result.
8. The frontend displays:
   - A plain-English risk summary explaining what was detected and why it matters.
   - Detected entity categories (e.g., EMAIL, PHONE, NAME, ADDRESS) with counts.
   - A redacted-text preview with replacement labels, and a one-click copy button.
   - A visible disclaimer: "Detection is not exhaustive. This is an aid, not a guarantee of anonymization or legal compliance."
9. No account record is created. The scan result is display-only.

### Authenticated path (logged-in user)

1. User clicks "Log in," enters credentials, and receives a JWT access token via `POST /api/token/`.
2. The UI stores the token in memory or sessionStorage (not localStorage, to reduce XSS exposure surface).
3. User pastes text (up to 10,000 characters) and presses "Scan."
4. The frontend sends `POST /api/v1/scans/` with a Bearer token.
5. The same detection, validation, and redaction pipeline runs.
6. On a cache hit: the cached sanitized result is returned; a history record is still created for this user.
7. On a cache miss: analysis runs, result is cached (if eligible), and a history record is created.
8. The frontend shows the same risk summary, entity categories, redacted preview, and copy button. A small badge indicates "from cache" when the result was served from Redis.
9. User navigates to the "History" tab. `GET /api/v1/scans/history/` returns their own records only — timestamp, risk level, detected categories, redacted preview, cache-hit status. Never raw submitted text.
10. User can see a detail view of any history record they own. They cannot reach any other user's records by ID manipulation or any other means.

### Repeat-scan / cache path

1. A user submits text identical (after normalization) to a previously cached eligible scan.
2. The backend computes the HMAC-SHA256 cache key and finds a valid entry in Redis.
3. The backend validates the cached entry's schema before returning it.
4. The full analysis pipeline is skipped; no Gemini call is made.
5. If the user is authenticated, a history record is created with `cache_hit: true`.
6. The response is structurally identical to a fresh scan result.

### Redis-unavailable path

1. User submits text. Redis connection fails or times out during the cache-lookup step.
2. The backend logs a non-sensitive operational error (no raw text in the log).
3. The backend proceeds with normal analysis — deterministic detection, Gemini call, redaction.
4. The result is returned to the user normally. No HTTP 500 is raised for a cache failure.
5. The backend may attempt to write the result to Redis after analysis; if that write also fails, it is silently ignored and the result is still returned.

---

## Screens and Layout

Source: `scope.md > Inspiration & Identity`.

### Single-page interface (one HTML file, Tailwind CSS via CDN)

**Scanner surface (default view)**
- Top: product name + brief tagline. Right: "Log in" link (unauthenticated) or username + "History" + "Log out" (authenticated).
- Center: large text area with a character counter. Below it: "Scan" button.
- Below the button: results panel (hidden until a scan completes), containing:
  - Risk summary card (plain-English explanation).
  - Entity category chips with counts.
  - Redacted text panel with copy button.
  - Disclaimer notice.
  - Cache-hit badge (authenticated only, shown when applicable).
- Error states appear inline below the button (validation errors, rate-limit notices, provider errors).

**Login panel** (modal or small inline form)
- Email/username and password fields.
- "Log in" submit button.
- Short note: access token is kept in session only.
- Error message inline on auth failure.

**History tab / view** (authenticated only)
- Tabular list: timestamp, risk level, detected categories, cache-hit indicator.
- Click a row → detail view showing the redacted preview and risk summary for that scan.
- "Back" link to return to scanner.
- Empty state: "No scan history yet. Run your first scan above."

---

## Look and Feel

Source: `scope.md > Inspiration & Identity`.

The tool should feel like a calm, professional developer instrument — the aesthetic of a linter or a code-quality checker, not a security-warning dashboard. Clean whitespace, neutral tones, clear typography. Entity-category chips can use muted color coding to distinguish types (e.g., EMAIL vs. NAME vs. PHONE) without being alarming. The redacted text preview uses a monospace font to signal "output you can inspect." No animations, no gamification. The risk summary is written in plain English, not legal language.

Font character: system-safe sans-serif or a clean CDN-loaded option (e.g., Inter). Monospace for the redacted preview.

---

## Features and Behavior

### US-1 — Public Instant Scan

*As an unauthenticated visitor, I want to submit text and receive a privacy analysis without creating an account.*

Endpoint: `POST /api/v1/scans/public/`

**Behavior:**
- Accept `{ "text": "<string>" }`. Reject on missing, null, empty string, or non-string values.
- Enforce maximum 2,000 characters. Reject before any model call.
- Apply deterministic detection (email, phone number) locally in Python. No Gemini call for pattern-detected types.
- Submit minimum necessary text to Gemini for contextual analysis (names, addresses, company-specific terms). Validate the structured JSON response.
- Merge local and model findings. Run deterministic redaction.
- Return structured result (see Response Schema).
- Do not create a scan-history record.
- Apply rate limit: 5 requests per minute per IP address. Return HTTP 429 when exceeded.
- Raw submitted text must not be written to the database, Redis, or application logs at any point.

**Acceptance criteria:**
- [ ] AC-US1-1: Valid text ≤ 2,000 chars receives HTTP 200 with well-formed result when analysis succeeds.
- [ ] AC-US1-2: Text exceeding 2,000 chars returns HTTP 400 before any Gemini call; test confirms no outbound request was made.
- [ ] AC-US1-3: Sixth request within one minute from the same IP returns HTTP 429.
- [ ] AC-US1-4: Missing or empty `text` field returns HTTP 400 with a stable machine-readable error code.
- [ ] AC-US1-5: Automated test confirms raw submitted text is absent from the database and logs after a completed scan.
- [ ] AC-US1-6: An unusable Gemini response returns a safe error (HTTP 502 or 503); no fabricated analysis is returned.
- [ ] AC-US1-7: The response and UI both include the product-limitation disclaimer.
- [ ] AC-US1-8: A valid scan with no detected PII returns HTTP 200 with an empty entities list and an appropriate risk summary.

---

### US-2 — Authenticated Scan and Persisted History

*As an authenticated user, I want to scan longer text and retrieve only my own scan history.*

Endpoints:
- `POST /api/v1/scans/` (authenticated scan)
- `GET /api/v1/scans/history/` (history list)
- `GET /api/v1/scans/history/{id}/` (history detail)
- `POST /api/token/` (obtain JWT)
- `POST /api/token/refresh/` (refresh JWT)

**Behavior:**
- Require a valid Bearer JWT for scan and history endpoints.
- Accept text up to 10,000 characters. Reject before any model call.
- Apply rate limit: 30 requests per minute per authenticated user.
- Reuse the identical detection, validation, and redaction pipeline as the public endpoint.
- On successful scan: create a history record containing — user reference, timestamp, risk level, detected categories and counts, redacted preview, cache-hit status, processing duration, scan schema version. No raw submitted text.
- `GET /api/v1/scans/history/` returns a paginated list of the authenticated user's records only.
- `GET /api/v1/scans/history/{id}/` returns a single record; returns HTTP 404 (not 403) if the record does not belong to the requesting user.
- Token refresh endpoint follows SimpleJWT defaults; does not expose user data in error responses.

**Acceptance criteria:**
- [ ] AC-US2-1: Request to `POST /api/v1/scans/` without a token returns HTTP 401.
- [ ] AC-US2-2: Valid authenticated scan ≤ 10,000 chars receives HTTP 200 and creates a history record containing no raw text field.
- [ ] AC-US2-3: Text exceeding 10,000 chars returns HTTP 400 before any Gemini call.
- [ ] AC-US2-4: `GET /api/v1/scans/history/` returns only records for the authenticated user; test confirms User A cannot see User B's records.
- [ ] AC-US2-5: `GET /api/v1/scans/history/{id}/` with another user's record ID returns HTTP 404.
- [ ] AC-US2-6: Automated test confirms no raw submitted text field exists in any history record in the database.
- [ ] AC-US2-7: Thirty-first request within one minute returns HTTP 429.
- [ ] AC-US2-8: Expired or invalid token returns HTTP 401 with no internal system details.

---

### US-3 — Duplicate Scan Caching

*As a user submitting the same text again, I want eligible results served from Redis without an additional Gemini call.*

**Behavior:**
- Before calling Gemini, compute an HMAC-SHA256 cache key from: normalized text + analysis_version + relevant configuration. Use a dedicated secret from environment configuration.
- If a valid cache entry exists and passes schema validation: return the cached result. Do not call Gemini.
- If the cache entry is missing, expired, or fails schema validation: run the full analysis pipeline, then attempt to cache the eligible sanitized result.
- Cache only results that: completed successfully, produced a valid validated response, and contain no raw text.
- Never cache: validation rejections, failed scans, provider errors, malformed model responses.
- TTL: 86,400 seconds (24 hours). Configurable via environment variable.
- Authenticated scans served from cache must still create a history record with `cache_hit: true`.
- Cache hits never bypass authentication or rate-limit checks.
- Public and authenticated scans share the cache namespace only where the result is demonstrably user-identity-independent. (For MVP: results are user-identity-independent because no user-specific context influences the analysis. Cache namespace is shared; history is always user-scoped.)

**Acceptance criteria:**
- [ ] AC-US3-1: Two identical eligible requests reuse a cache entry; a test with a mocked Gemini client confirms the model is called exactly once.
- [ ] AC-US3-2: Cache keys are HMAC-SHA256 hex digests; they contain no substring of the raw submitted text.
- [ ] AC-US3-3: Cache entries expire after 86,400 seconds; TTL is set correctly on write.
- [ ] AC-US3-4: A configuration change that increments `analysis_version` causes a cache miss for previously cached results.
- [ ] AC-US3-5: A failed or invalid scan result is not written to the cache; test confirms no Redis SET is called for a failed scan.
- [ ] AC-US3-6: An authenticated cache-hit scan creates a history record marked `cache_hit: true`.
- [ ] AC-US3-7: A cached result fails schema validation and is discarded; the pipeline falls back to a fresh analysis.

---

### US-4 — Graceful Redis Fallback

*As a user, I want the scanner to remain usable when Redis is temporarily unavailable.*

**Behavior:**
- All Redis operations (GET, SET) are wrapped in exception handling. A failure at either step does not raise HTTP 500.
- On cache GET failure: log a non-sensitive operational error; proceed with full analysis.
- On cache SET failure: log a non-sensitive operational error; return the scan result normally.
- Log messages must not contain raw submitted text, cache key material that reveals text, secrets, or stack traces with sensitive data.
- If Gemini is unavailable or returns an unusable response: return a safe error response. Never fabricate analysis output.

**Acceptance criteria:**
- [ ] AC-US4-1: With Redis stopped, a valid scan completes with HTTP 200 (when Gemini is available).
- [ ] AC-US4-2: A simulated cache-GET exception does not crash the scan endpoint; it falls through to analysis.
- [ ] AC-US4-3: A simulated cache-SET exception does not invalidate a successful scan result; HTTP 200 is returned.
- [ ] AC-US4-4: Captured log output during Redis failure contains no raw submitted text.
- [ ] AC-US4-5: A simulated Gemini failure or malformed response returns HTTP 502; the response body is a structured error with no fabricated risk data.

---

## API Contracts

### Request: Public Scan

```
POST /api/v1/scans/public/
Content-Type: application/json

{
  "text": "<string, 1–2000 characters>"
}
```

### Request: Authenticated Scan

```
POST /api/v1/scans/
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "text": "<string, 1–10000 characters>"
}
```

### Successful Scan Response (both endpoints)

```json
{
  "scan_id": "<uuid | null for public>",
  "status": "completed",
  "cached": false,
  "risk_level": "high | medium | low | none",
  "risk_summary": "<plain-English explanation>",
  "entities": [
    {
      "category": "EMAIL | PHONE | NAME | ADDRESS",
      "count": 2,
      "source": "deterministic | model | both"
    }
  ],
  "redacted_text": "<text with [EMAIL], [PHONE], [NAME], [ADDRESS] labels>",
  "disclaimer": "Detection is not exhaustive. This is an aid, not a guarantee of anonymization or legal compliance.",
  "analysis_version": "<string>",
  "processing_ms": 412
}
```

When `cached: true`, `processing_ms` reflects cache-lookup time, not model-analysis time.

### Scan History List Response

```
GET /api/v1/scans/history/
Authorization: Bearer <access_token>
```

```json
{
  "count": 12,
  "next": "/api/v1/scans/history/?page=2",
  "previous": null,
  "results": [
    {
      "scan_id": "<uuid>",
      "timestamp": "2026-10-16T09:00:00Z",
      "risk_level": "high",
      "entity_categories": ["EMAIL", "NAME"],
      "cached": false,
      "redacted_preview": "<first 200 chars of redacted_text>"
    }
  ]
}
```

### Scan History Detail Response

```
GET /api/v1/scans/history/{scan_id}/
Authorization: Bearer <access_token>
```

Returns the full scan result fields (same shape as the scan response) minus `text`. Returns HTTP 404 if the record does not exist or does not belong to the requesting user.

### Error Response Schema (all endpoints)

```json
{
  "error": {
    "code": "<MACHINE_READABLE_CODE>",
    "message": "<human-readable, safe for display>",
    "details": {}
  }
}
```

### HTTP Status Codes and Machine-Readable Error Codes

| HTTP Status | Code constant | When used |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Missing, empty, or invalid `text` field; malformed JSON |
| 400 | `TEXT_TOO_LONG` | Input exceeds the applicable character limit |
| 401 | `AUTHENTICATION_REQUIRED` | Missing, expired, or invalid JWT |
| 403 | `PERMISSION_DENIED` | Authenticated but not authorized for the resource (reserved; history uses 404) |
| 404 | `NOT_FOUND` | History record does not exist or belongs to another user |
| 405 | `METHOD_NOT_ALLOWED` | Unsupported HTTP method on an endpoint |
| 429 | `RATE_LIMIT_EXCEEDED` | Requests exceed the applicable rate limit |
| 500 | `INTERNAL_ERROR` | Unexpected server error; no internal details exposed |
| 502 | `PROVIDER_ERROR` | Gemini returned an unusable or invalid response |
| 503 | `SERVICE_UNAVAILABLE` | A required dependency is unavailable and the scan cannot proceed |

HTTP 500 must not be returned for a Redis cache failure when the scan can otherwise complete.
HTTP 429 responses must include a `Retry-After` header where feasible.

---

## Configuration and System Defaults

Source: `scope.md > The POC Boundary`.

| Setting | Default | Configurable |
|---|---|---|
| Public endpoint | `POST /api/v1/scans/public/` | No |
| Public maximum text length | 2,000 characters | Via env var |
| Public rate limit | 5 scans / minute / IP | Via env var |
| Authenticated endpoint | `POST /api/v1/scans/` | No |
| Authenticated maximum text length | 10,000 characters | Via env var |
| Authenticated rate limit | 30 scans / minute / user | Via env var |
| History endpoint | `GET /api/v1/scans/history/` | No |
| Cache TTL | 86,400 seconds (24 hours) | Via env var |
| Cache-key digest | HMAC-SHA256 with dedicated env-supplied secret | Secret via env var only |
| AI provider | Gemini (primary) | Via env var (model name) |
| Database | SQLite (local dev) | Via env var (path / future DSN) |
| Frontend | Static HTML, Tailwind CDN | No framework |
| Orchestration | Docker Compose | — |

**Rate limiting implementation:** Use Django REST Framework's built-in throttling (`AnonRateThrottle` for IP-based, `UserRateThrottle` for user-based). For IP determination, use `REMOTE_ADDR` from `request.META` by default; document how to configure `TRUSTED_PROXIES` or `USE_X_FORWARDED_FOR` safely if the stack is placed behind a reverse proxy. Never blindly trust `X-Forwarded-For` without explicit configuration.

**Request-size validation:** Character-count validation happens in the DRF serializer `validate_text()` method, before any model query, Redis call, or Gemini request. Do not rely on body-size limits alone.

**Secrets:** `DJANGO_SECRET_KEY`, `CACHE_HMAC_SECRET`, `GEMINI_API_KEY` are mandatory environment variables. The application must refuse to start (or log a clear fatal error) if required secrets are absent. No secret may appear in `settings.py`, source code, or commit history. `.env.example` contains placeholder strings only.

---

## Privacy and Security Invariants

Source: `scope.md > Privacy invariants`.

These are verified requirements, not statements of intent. Each must have a corresponding automated test.

### INV-1 — Zero Persistence of Raw Submitted Text

The original submitted text must never be written to:
- Any SQLite table column (including debug columns, log tables, or serializer output).
- Any Redis key or value.
- Any application log line (Django logger, print statements, exception handlers, Sentry-style error reporters).
- Any scan-history record field.

In-memory processing is permitted for the duration of a single request. Exception handlers, DRF serializers, and the Gemini client wrapper must be explicitly reviewed to ensure they do not capture and log request bodies.

*Test:* Submit a scan. Query all database tables. Search log output. Assert the submitted string is absent from both.

### INV-2 — Safe Cache Design

- Cache keys are HMAC-SHA256 hex digests derived from normalized text, `analysis_version`, and relevant configuration using a dedicated secret.
- The cache secret is an environment variable only; it is never committed or hardcoded.
- If the cache secret is missing or empty at startup, the application raises a configuration error; it does not fall back to an insecure plain-hash key.
- Cache values contain only validated sanitized output — redacted text, risk summary, entity categories, risk level — and must not contain raw input text or raw provider responses.
- Cached values are treated as untrusted: schema validation runs before the value is used.

### INV-3 — Validate Gemini Structured Output

The Gemini prompt requests a structured JSON response with a documented schema. The backend:
1. Parses the raw model response as JSON (handles `JSONDecodeError` safely).
2. Validates required fields: `risk_level` (enum), `risk_summary` (non-empty string, max length), `entities` (list of objects with `category` and `text` fields), optionally `spans`.
3. Validates each entity: `category` must be one of `EMAIL | PHONE | NAME | ADDRESS`; text must be a non-empty string.
4. Rejects responses that are missing required fields, contain unknown category values, or exceed defined length bounds.
5. If validation fails: returns a safe error; does not use partial model output for redaction.
6. Span positions (if used): validated against the original text before redaction is applied.

Deterministic redaction is applied by server code using Python `str.replace()` or span-based logic — never by asking the model to rewrite the text.

### INV-4 — Product Limitation Disclosure

Every scan response (HTTP 200) must include a `disclaimer` field. The UI must display the disclaimer text prominently adjacent to the results panel. The disclaimer must include:
- Detection may miss sensitive information.
- Redaction does not guarantee full anonymization.
- This tool does not guarantee legal or regulatory compliance.
- Users remain responsible for reviewing output before sharing with external services.

The product must not use the phrases "100% anonymization," "completely safe," "guaranteed PII-free," or equivalent.

### INV-5 — External AI Processing Disclosure

The UI must display, before the user submits their first scan, a notice that text may be sent to the configured Gemini service for contextual analysis. This notice must be visible (not buried in a terms document). It does not require a click-through; it must be legible on the scanner surface.

The API response must indicate whether external analysis was performed (derivable from `cached: true/false` and the entities' `source` field).

---

## States and Boundaries

- **First use (unauthenticated):** Scanner is immediately usable. No account prompt on load. The external-AI disclosure notice is visible on the scanner surface.
- **Empty input:** "Scan" button is disabled or returns HTTP 400; no model call is made.
- **Text too long:** Inline character counter goes red; submit returns HTTP 400 with `TEXT_TOO_LONG`; no model call.
- **Rate limit exceeded:** HTTP 429 returned; UI shows a friendly "Too many requests, please wait" message.
- **Valid scan, no PII detected:** HTTP 200; `entities: []`; risk level `none`; risk summary explains that no supported patterns were found and reiterates detection limitations. Not an error state.
- **Gemini unavailable or invalid response:** HTTP 502 or 503; UI shows "Analysis unavailable — please try again shortly." No fabricated result.
- **Redis unavailable:** Scan proceeds normally; no user-visible error (Redis is an optimization). Operational log entry (no sensitive data).
- **Cache hit:** Response identical in shape to a fresh scan; `cached: true`; UI shows a small badge.
- **Authenticated, history empty:** "No scan history yet" empty state.
- **History record not owned by requesting user:** HTTP 404 (not 403, to avoid confirming record existence).
- **Session expired:** JWT expiry returns HTTP 401; UI prompts re-login without losing the current text input.

---

## Product Decisions

- **HTML/JS + Tailwind CDN (no framework):** Keeps engineering effort on the backend and privacy pipeline; sufficient to demonstrate the workflow on screen.
- **Public scanning included:** Allows immediate demonstration without account creation; rate-limited to limit abuse.
- **Gemini as primary provider:** Gemini Flash is the initial model; exact model identifier confirmed before implementation. Provider abstraction allows a future OpenAI adapter without touching the scan pipeline.
- **HMAC-SHA256 cache keys:** Keyed construction prevents raw-text exposure in cache storage; a plain hash would not provide the keying security property.
- **HTTP 404 (not 403) for unauthorized history records:** Avoids confirming the existence of another user's record.
- **Token stored in sessionStorage (not localStorage):** Reduces persistent XSS exposure; acceptable for a demo MVP.
- **SQLite for MVP:** Straightforward local setup; schema will be designed to migrate cleanly to PostgreSQL later.
- **Rate limiting via DRF throttling classes:** Supported, documented, and testable without an additional dependency.

---

## What We're Building

The proof of concept is complete when all of the following are true:

- [ ] Public scan: valid text → risk explanation + redacted preview in the browser.
- [ ] Input-size enforcement: 2,000-char public limit, 10,000-char auth limit, enforced before any model call.
- [ ] Public rate limit: 5/min/IP; sixth request returns HTTP 429.
- [ ] Authenticated rate limit: 30/min/user.
- [ ] JWT auth: obtain and refresh tokens; protected endpoints return 401 without a valid token.
- [ ] Scan history: authenticated user sees their own records; cannot access another user's records.
- [ ] Raw-text absence: automated test confirms no raw submitted text in the database or logs after a scan.
- [ ] Cache hit: identical eligible scan reuses Redis result; Gemini is called exactly once (test with mock).
- [ ] Cache TTL: Redis entry has TTL of 86,400 seconds (verified in a test).
- [ ] Redis fallback: scan completes when Redis is down; no HTTP 500 for cache failure alone.
- [ ] Invalid Gemini output: malformed/missing model response returns HTTP 502; no fabricated result.
- [ ] Limitation disclaimer: every scan response and the UI include the required disclaimer.
- [ ] External-AI disclosure: visible on the scanner surface before the user submits text.
- [ ] Docker Compose: `docker compose up` starts frontend, backend, and Redis; the demo works end to end.
- [ ] `.env.example`: all required environment variables documented with placeholder values.
- [ ] Tests: unit and API tests cover detection, redaction, auth, authorization, cache behavior, and Redis failure.

---

## Deferred From the POC

- OpenAI adapter — provider abstraction is designed for it; implementation deferred until core workflow is complete.
- PostgreSQL — schema migration path is kept clean; SQLite for the demo.
- User-managed cache invalidation — "clear my cached scans" is a later feature.

---

## Possible Later Enhancements

Source: `scope.md > Later`.

- Additional detection categories: API keys, IBANs, national ID number formats.
- Per-user rate-limit configuration.
- Admin analytics dashboard.
- File upload support (PDF, DOCX, CSV) — a separate engineering problem.

---

## Non-Goals

Source: `scope.md > Explicitly Cut`.

- **React/Vue/Next.js frontend** — unnecessary build complexity for this proof of concept.
- **Full OpenAI provider implementation** — provider abstraction is built; a second full integration is not.
- **Multiple document formats** — text input only; file parsing is out of scope.
- **Enterprise access controls** — no teams, roles, or org-level history in this version.
- **Production deployment** — Docker Compose for local demo only; no cloud hosting, TLS termination, or managed Redis.
- **Legal compliance guarantees** — explicitly excluded as a product promise; the application must say so.
- **Perfect PII detection** — the product actively disclaims exhaustive detection.

---

## Open Questions

| # | Question | Blocks spec? |
|---|---|---|
| OQ-1 | Which specific Gemini model identifier is currently available and appropriate for the demo? (`gemini-1.5-flash` was proposed; confirm against the API before the spec is finalized.) | Yes — needed before the Gemini integration section of `4-spec`. |
| OQ-2 | Should the frontend be served by a dedicated Nginx container, by Django (via `collectstatic`), or by a minimal Python HTTP server in Docker Compose? | Yes — affects the Docker Compose service definition in `4-spec`. |
| OQ-3 | Should `analysis_version` be a manually bumped constant in `settings.py`, or derived automatically from a hash of the detection/redaction configuration? | No — can default to a manually bumped version string for the MVP; revisit in spec. |
| OQ-4 | What is the canonical normalization step applied before HMAC key derivation? (e.g., Unicode NFC normalization + strip leading/trailing whitespace + collapse internal whitespace to single spaces.) | No — can be decided and documented in `4-spec`. |
