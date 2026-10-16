---
doc: spec
status: draft
---

# Privacy Copilot for AI Workflows — Technical Spec

## How This Works, In Plain Language

The application is a single Django web server that does three things: serves the HTML frontend, exposes a REST API for scanning text, and manages user accounts and scan history.

When you paste text and press "Scan," your browser sends it to Django. Django first runs its own Python code to look for common patterns like email addresses and phone numbers — this happens entirely on the server, with no external call. It then sends the minimum needed context to Google Gemini to check for harder-to-spot things like people's names or company references. Gemini replies with a structured list of what it found. Django checks that list carefully (because model output can be wrong), merges it with what it found locally, and then replaces each sensitive item with a label like `[EMAIL]` or `[NAME]` — using its own Python code, not by asking Gemini to rewrite the text.

Redis sits alongside Django as a speed optimization. The first time a piece of text is scanned, Django saves the sanitized result in Redis using a keyed HMAC digest of the text — so the text itself never appears in the key or value, and is never stored anywhere. The next time the same text is submitted, Django finds that fingerprint in Redis and skips the Gemini call. If Redis is down, Django continues normally; it just skips the cache step.

Authenticated users hold a JWT in their browser session. Protected endpoints check that token before acting. Each authenticated scan creates a history record containing only the sanitized result and metadata — never the original text.

The whole thing runs with two `docker compose up` services: the Django application (which also serves the frontend) and Redis.

---

## The Core Journey Through the System

Implements `prd.md > The Core Journey`.

```
PUBLIC SCAN (cache miss):
Browser → POST /api/v1/scans/public/ {text}
       → Django: validate input (size, type)
       → Django: deterministic regex detection (local, no I/O)
       → Django: compute HMAC-SHA256 cache key
       → Redis: GET key → miss
       → Gemini: analyze(text) → structured JSON
       → Django: validate model response schema
       → Django: merge findings, deterministic redact
       → Redis: SET sanitized result, TTL 86400s
       → Browser: 200 {redacted_text, risk_level, entities, disclaimer}

CACHE HIT (authenticated):
Browser → POST /api/v1/scans/ + Bearer token
       → Django: validate JWT, check rate limit, validate input
       → Django: compute HMAC cache key
       → Redis: GET key → hit
       → Django: validate cached schema
       → Django: create ScanRecord (cache_hit=True)
       → Browser: 200 {redacted_text, ..., cached: true}

REDIS UNAVAILABLE:
Step at Redis GET raises RedisError
       → Django: log non-sensitive operational warning (no raw text)
       → Django: continue to Gemini call
       → Django: complete scan normally
       → Redis SET attempt also fails → log, ignore
       → Browser: 200 (scan succeeds; cache miss implied)

GEMINI UNAVAILABLE / INVALID RESPONSE:
Redis miss → Gemini call fails or returns unparseable JSON
       → Django: raise ProviderResponseError
       → Django: return HTTP 502 PROVIDER_ERROR (no fabricated result)
       → Browser: shows "Analysis unavailable"

HISTORY RETRIEVAL:
Browser → GET /api/v1/scans/history/ + Bearer token
       → Django: validate JWT
       → Django: ScanRecord.objects.filter(user=request.user).order_by('-created_at')
       → Browser: paginated list of sanitized scan records
```

---

## Stack

| Layer | Choice | Rationale |
|---|---|---|
| Language | Python 3.11+ | Learner's primary language |
| Web framework | Django 4.2 LTS + DRF 3.15 | Learner has hands-on DRF/SimpleJWT experience; LTS stability |
| Authentication | `djangorestframework-simplejwt` 5.x | Learner specifically trained on this |
| AI provider | `google-generativeai` SDK | Agreed primary provider; behind abstraction interface |
| Gemini model | `gemini-1.5-flash` (env: `GEMINI_MODEL`) | Fast Flash tier; env-configurable; verify availability on first test |
| Cache | Redis 7 + `redis-py` 5.x | Agreed caching layer |
| Static files | WhiteNoise 6.x | Serves frontend from Django; no separate container |
| Frontend | HTML5 + vanilla JS + Tailwind CSS v3 CDN | Agreed; no build step or framework |
| Rate limiting | DRF `AnonRateThrottle` + `UserRateThrottle` | Built-in; no extra dependency |
| Containerization | Docker + Docker Compose v2 | Agreed local orchestration |
| Testing | `pytest-django` + `unittest.mock` | Idiomatic Django testing; mock is stdlib |
| HMAC | Python stdlib `hmac` + `hashlib` | No extra dependency |
| WSGI server | `gunicorn` | Standard Django production WSGI server |

**Documentation links:**
- Django 4.2: https://docs.djangoproject.com/en/4.2/
- DRF: https://www.django-rest-framework.org/
- SimpleJWT: https://django-rest-framework-simplejwt.readthedocs.io/
- google-generativeai: https://ai.google.dev/gemini-api/docs/quickstart?lang=python
- redis-py: https://redis-py.readthedocs.io/
- WhiteNoise: https://whitenoise.readthedocs.io/
- pytest-django: https://pytest-django.readthedocs.io/
- Gemini rate limits: https://ai.google.dev/gemini-api/docs/rate-limits

> **Verify before first integration test:** confirm `gemini-1.5-flash` is available under the configured API key. If not, change `GEMINI_MODEL` in `.env` (e.g., to `gemini-2.0-flash`) — no code changes required.

---

## Where It Runs and How Someone Tries It

**Runtime:** Local machine via Docker Compose. No cloud hosting required.

**Requirements:** Docker Desktop, a Gemini API key, `.env` file populated from `.env.example`.

**Start command:**
```bash
cp .env.example .env   # fill in GEMINI_API_KEY and secrets
docker compose up --build
```

**Open:** `http://localhost:8000`

**Demo sequence (for video recording):**
1. Open `http://localhost:8000`. Note the external-AI disclosure and disclaimer visible on the scanner surface.
2. Paste text containing a name, email, and phone number. Press "Scan." Show risk summary, entity chips, redacted preview, and disclaimer.
3. Log in with test credentials. Scan the same text. Show the "from cache" badge and the history entry.
4. Open an incognito tab, log in as a second user. Confirm history is empty (ownership isolation).
5. `docker compose stop redis`. Scan again. Show the scan completes — Redis is optional.
6. `docker compose exec web pytest tests/ -v` — show the test suite passing.

**Submission:** short demo video (recorded locally) + public GitHub repository with README setup instructions. Deployment is optional.

---

## Look and Feel

Implements `prd.md > Look and Feel`, `scope.md > Inspiration & Identity`.

- **Palette:** Light neutral background; dark text. Muted per-category chip colours: teal for EMAIL, amber for PHONE, slate-blue for NAME, rose for ADDRESS. No alarm-red.
- **Typography:** Inter via CDN; `font-mono` for the redacted-text preview panel.
- **Density:** Spacious, single-column. Scanner area takes most of the viewport.
- **Tone:** Plain English. Risk summary example: "We found 2 email addresses and 1 phone number. Sharing this with an external AI service could expose personal contact details."
- **Implementation:** Tailwind CDN `<script src="https://cdn.tailwindcss.com">` in `<head>`. No PostCSS. All text set via `textContent` (never `innerHTML`).

---

## Components

### 1. Django Project Configuration

**Files:** `privacy_copilot/settings.py`, `privacy_copilot/urls.py`

- `INSTALLED_APPS`: `rest_framework`, `rest_framework_simplejwt`, `whitenoise`, `scanner`.
- `MIDDLEWARE`: `WhiteNoiseMiddleware` after `SecurityMiddleware`.
- `STATIC_ROOT` + `STATICFILES_DIRS`: configured for `collectstatic` to pull `frontend/` assets.
- `REST_FRAMEWORK`: default auth `JWTAuthentication`; default throttle classes and rates.
- All secrets loaded from environment; `ImproperlyConfigured` raised at startup if `CACHE_HMAC_SECRET` or `DJANGO_SECRET_KEY` are absent/empty.
- Root URL conf: `/api/v1/` → scanner app; `/api/token/` → SimpleJWT; catch-all GET → `index.html`.

PRD ref: `prd.md > Configuration and System Defaults`.

---

### 2. Frontend (Static HTML/JS)

**Files:** `frontend/index.html`, `frontend/app.js`

Served by WhiteNoise from `STATIC_ROOT`. Catch-all URL returns `index.html` for any non-API path.

- **Scanner surface:** text area with live character counter (red when over limit); "Scan" button; external-AI disclosure (always visible before first scan); results panel (hidden until response); disclaimer shown with results.
- **Login panel:** modal or inline form; `POST /api/token/`; stores `access` token in `sessionStorage['pc_access_token']`.
- **Authenticated nav:** username, "History" tab, "Log out".
- **History tab:** `GET /api/v1/scans/history/`; table with timestamp, risk level, categories, cache-hit indicator; row click → detail view.
- **Cache-hit badge:** shown when `cached: true` in response.
- **Safe rendering:** all dynamic content via `textContent`/`innerText`. No `innerHTML` for user-supplied data.
- **Error handling:** inline messages for 400, 401, 429, 502, 503. On 401, clear token and prompt login without losing textarea content.
- **No API keys in JS.**

PRD ref: `prd.md > Screens and Layout`, `prd.md > States and Boundaries`.

---

### 3. Scanner API Views

**File:** `scanner/views.py`

#### `PublicScanView` — `POST /api/v1/scans/public/`
- `permission_classes = [AllowAny]`
- `throttle_classes = [PublicScanThrottle]`
- Flow: `ScanRequestSerializer(max_length=PUBLIC_MAX_TEXT_LENGTH)` → `ScanService.execute(text)` → `ScanResultSerializer` → HTTP 200
- Does **not** create a `ScanRecord`.

#### `AuthenticatedScanView` — `POST /api/v1/scans/`
- `permission_classes = [IsAuthenticated]`
- `throttle_classes = [UserScanThrottle]`
- Flow: `ScanRequestSerializer(max_length=AUTH_MAX_TEXT_LENGTH)` → `ScanService.execute(text)` → create `ScanRecord` → HTTP 200

#### `ScanHistoryListView` — `GET /api/v1/scans/history/`
- `permission_classes = [IsAuthenticated]`
- `ScanRecord.objects.filter(user=request.user).order_by('-created_at')`
- `PageNumberPagination`, 20 per page.

#### `ScanHistoryDetailView` — `GET /api/v1/scans/history/{scan_id}/`
- `permission_classes = [IsAuthenticated]`
- `get_object_or_404(ScanRecord, scan_id=scan_id, user=request.user)` — returns 404 for missing and other-user records.

PRD ref: `prd.md > US-1`, `prd.md > US-2`.

---

### 4. DRF Serializers and Request Validation

**File:** `scanner/serializers.py`

`ScanRequestSerializer`: `text = CharField(required=True, allow_blank=False, max_length=<limit>)`. Validation in `validate_text()` fires before any service call. Max length injected by view.

`ScanResultSerializer`: read-only; serializes `ScanResult` dataclass into agreed response shape. All fields explicitly typed.

`ScanRecordSerializer` (history): explicit `Meta.fields` allowlist — never `'__all__'`. No field that could contain raw text.

Custom `exception_handler` in `scanner/exceptions.py` wraps all DRF errors into `{"error": {"code": ..., "message": ..., "details": ...}}`.

PRD ref: `prd.md > API Contracts`, `prd.md > HTTP Status Codes`.

---

### 5. ScanService

**File:** `scanner/services/scan_service.py`

Central orchestrator — stateless, called by both views.

```
ScanService.execute(text: str) -> ScanResult
  1. DetectionService.detect(text)           → local_detections
  2. CacheService.make_key(text, version, cfg) → cache_key
  3. CacheService.get(cache_key)             → cached_result or None
  4. If cached_result: return ScanResult(cached=True, ...)
  5. GeminiProvider.analyze(text)            → raw_model_response
  6. ResponseValidator.validate(raw)         → model_detections or raises ProviderResponseError
  7. DetectionService.merge(local, model)    → merged_detections
  8. RedactionService.redact(text, merged)   → redacted_text
  9. Build ScanResult
  10. CacheService.set(cache_key, result, ttl)  [silent on error]
  11. return ScanResult
```

PRD ref: `prd.md > US-1`, `prd.md > US-3`, `prd.md > US-4`.

---

### 6. DetectionService

**File:** `scanner/services/detection.py`

- Regex detection for `EMAIL` (RFC 5322-simplified pattern) and `PHONE` (E.164 + common US/international formats).
- Returns `List[Detection(category, start, end, matched_text, source)]`.
- `merge(local, model)`: sort by `start`; for overlapping spans prefer `deterministic` over `model`; drop fully-contained shorter spans.
- Validates model detections: category must be in `{EMAIL, PHONE, NAME, ADDRESS}`; matched text non-empty string; invalid detections dropped (non-sensitive log, no raw text).

PRD ref: `prd.md > Features and Behavior > US-1`, `prd.md > INV-3`.

---

### 7. RedactionService

**File:** `scanner/services/redaction.py`

```
RedactionService.redact(text: str, detections: List[Detection]) -> str
```

- Sort detections by `start` descending (right-to-left preserves earlier offsets).
- Span-based: validate `0 <= start < end <= len(text)`; skip invalid with non-sensitive log warning.
- Replacement labels: `[EMAIL]`, `[PHONE]`, `[NAME]`, `[ADDRESS]`.
- Returns redacted string. Never modifies input in-place.

PRD ref: `prd.md > Features and Behavior > US-1`.

---

### 8. GeminiProvider

**Files:** `scanner/providers/base.py`, `scanner/providers/gemini_provider.py`

```python
# base.py
class PrivacyAnalysisProvider(ABC):
    @abstractmethod
    def analyze(self, text: str) -> dict: ...
```

`GeminiProvider`:
- Loads `GEMINI_API_KEY`, `GEMINI_MODEL` (default `gemini-1.5-flash`), `GEMINI_TIMEOUT_SECONDS` (default 20) from env.
- Builds structured prompt requesting JSON: `{risk_level, risk_summary, entities: [{category, text, reason}]}`.
- Calls Gemini SDK with `response_mime_type="application/json"`.
- On `JSONDecodeError` or `GoogleAPIError`: raises `ProviderResponseError` with non-sensitive message.
- Returns raw parsed dict to `ResponseValidator`. Does not apply redaction.
- **To switch models:** change `GEMINI_MODEL` env var. No code changes.
- **Future provider:** implement `PrivacyAnalysisProvider` in a new file; factory function reads `AI_PROVIDER` env var.

PRD ref: `prd.md > INV-3`, `prd.md > INV-5`.

---

### 9. ResponseValidator

**File:** `scanner/services/response_validator.py`

Strict schema validation before any field is used:
- Required top-level: `risk_level` (one of `high|medium|low|none`), `risk_summary` (non-empty string, max 1000 chars), `entities` (list).
- Per entity: `category` (one of `EMAIL|PHONE|NAME|ADDRESS`), `text` (non-empty string, max 500 chars).
- Top-level failure → raises `ProviderResponseError` (no partial use).
- Per-entity failure → drop that entity; log non-sensitive warning; continue with valid entities.

PRD ref: `prd.md > INV-3`.

---

### 10. CacheService

**File:** `scanner/services/cache_service.py`

**Key derivation:**
```python
import hmac, hashlib, json, unicodedata

def make_key(text, analysis_version, config):
    secret = settings.CACHE_HMAC_SECRET.encode()
    normalized = " ".join(unicodedata.normalize("NFC", text).split())
    payload = json.dumps(
        {"text": normalized, "v": analysis_version, "cfg": config},
        sort_keys=True, ensure_ascii=False, separators=(',', ':')
    )
    digest = hmac.new(secret, payload.encode('utf-8'), hashlib.sha256).hexdigest()
    return f"pcscan:{digest}"
```

- `analysis_version`: string constant `"v1"` in `settings.py`; bump when detection or redaction labels change incompatibly.
- `config`: `{"model": GEMINI_MODEL}` (extend if other result-affecting settings are added).
- Raw text never in the key string.

**Cached value schema (JSON):**
```json
{
  "schema_version": "1",
  "risk_level": "...",
  "risk_summary": "...",
  "entities": [...],
  "redacted_text": "...",
  "disclaimer": "..."
}
```
No raw text, no provider response, no user data.

**Operations** (both wrapped in `try/except redis.RedisError`):
- `get(key)`: deserialize → validate schema → return `ScanResult` or `None`.
- `set(key, result, ttl)`: `redis_client.setex(key, ttl, json_value)`. Silent on error.

TTL: `settings.CACHE_TTL` (default 86400, from `CACHE_TTL` env var).

PRD ref: `prd.md > US-3`, `prd.md > US-4`, `prd.md > INV-1`, `prd.md > INV-2`.

---

### 11. ScanRecord Model

**File:** `scanner/models.py`

```python
class ScanRecord(models.Model):
    scan_id            = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    user               = models.ForeignKey(settings.AUTH_USER_MODEL,
                                           on_delete=models.CASCADE, related_name='scan_records')
    created_at         = models.DateTimeField(auto_now_add=True, db_index=True)
    risk_level         = models.CharField(max_length=10,
                                          choices=[('high','High'),('medium','Medium'),
                                                   ('low','Low'),('none','None')])
    risk_summary       = models.TextField()
    entity_categories  = models.JSONField(default=list)   # ["EMAIL","NAME"]
    entity_counts      = models.JSONField(default=dict)   # {"EMAIL":2,"NAME":1}
    redacted_preview   = models.TextField()               # first 500 chars of redacted_text
    full_redacted_text = models.TextField()
    cache_hit          = models.BooleanField(default=False)
    processing_ms      = models.IntegerField(null=True)
    analysis_version   = models.CharField(max_length=20)
    schema_version     = models.CharField(max_length=10, default='1')

    class Meta:
        ordering = ['-created_at']
        indexes = [models.Index(fields=['user', '-created_at'])]
```

**Non-negotiable:** no `raw_text`, `submitted_text`, `input_text`, or equivalent field. History queries always `filter(user=request.user)`.

PRD ref: `prd.md > US-2`.

---

### 12. Authentication and JWT

**Files:** `privacy_copilot/urls.py`, `scanner/urls.py`

```
POST /api/token/          → TokenObtainPairView (SimpleJWT default)
POST /api/token/refresh/  → TokenRefreshView (SimpleJWT default)
```

SimpleJWT settings:
```python
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=60),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "AUTH_HEADER_TYPES": ("Bearer",),
}
```

Frontend: store `access` token in `sessionStorage['pc_access_token']`; send as `Authorization: Bearer <token>`; on 401, clear and prompt login.

PRD ref: `prd.md > US-2`.

---

### 13. Rate Limiting

**File:** `scanner/throttling.py`

```python
class PublicScanThrottle(AnonRateThrottle):
    rate = '5/min'       # overridable via settings.DEFAULT_THROTTLE_RATES

class UserScanThrottle(UserRateThrottle):
    rate = '30/min'
```

`AnonRateThrottle` uses `REMOTE_ADDR`. For reverse proxy setups (not required for MVP), configure `SECURE_PROXY_SSL_HEADER` and a trusted proxy list — never blindly trust `X-Forwarded-For`. Document in README.

PRD ref: `prd.md > Configuration and System Defaults`, `prd.md > INV public rate limit`.

---

### 14. Privacy and Logging

**File:** `privacy_copilot/settings.py` (logging section)

- Django logging → stdout (Docker captures).
- Log formatter explicitly excludes request body.
- All exception handlers strip raw-text content before logging.
- Zero-persistence invariant test: submit a known sentinel text → scan → assert sentinel absent from all `ScanRecord` field values and from captured `caplog` output.

PRD ref: `prd.md > INV-1`.

---

## Data Model

Implements `prd.md > US-2`.

| Data | Lives in | Persists? |
|---|---|---|
| Submitted text | In-memory only, one request | Never written anywhere |
| JWT access token | Browser `sessionStorage` | No — clears on tab close |
| Scan result (public) | HTTP response only | No |
| Scan result (authenticated) | `ScanRecord` in SQLite | Yes |
| Cached sanitized result | Redis (HMAC key, 24h TTL) | Until TTL expires |
| User account | Django `auth_user` table | Yes |

---

## File Structure

```
privacy-copilot-api/
|
+-- privacy_copilot/              # Django project package
|   +-- __init__.py
|   +-- settings.py               # All config; secrets from env vars
|   +-- urls.py                   # Root URLs + catch-all for frontend SPA
|   +-- wsgi.py
|
+-- scanner/                      # Main Django app
|   +-- __init__.py
|   +-- admin.py
|   +-- apps.py                   # AppConfig; checks CACHE_HMAC_SECRET on ready()
|   +-- models.py                 # ScanRecord
|   +-- serializers.py            # Request + response + history serializers
|   +-- views.py                  # PublicScanView, AuthenticatedScanView, history views
|   +-- urls.py                   # /api/v1/ routes
|   +-- throttling.py             # PublicScanThrottle, UserScanThrottle
|   +-- exceptions.py             # Custom DRF exception handler
|   +-- migrations/
|   |   +-- 0001_initial.py
|   |
|   +-- services/
|   |   +-- __init__.py
|   |   +-- scan_service.py       # ScanService orchestrator
|   |   +-- detection.py          # DetectionService (regex + merge)
|   |   +-- redaction.py          # RedactionService (span-based replacement)
|   |   +-- cache_service.py      # CacheService (HMAC key, Redis get/set)
|   |   +-- response_validator.py # ResponseValidator (Gemini output schema)
|   |
|   +-- providers/
|       +-- __init__.py
|       +-- base.py               # PrivacyAnalysisProvider ABC
|       +-- gemini_provider.py    # GeminiProvider implementation
|
+-- frontend/
|   +-- index.html                # SPA shell + Tailwind CDN link
|   +-- app.js                    # Vanilla JS: scan, login, history, errors
|
+-- tests/
|   +-- conftest.py               # Fixtures: test client, mock provider, test users
|   +-- test_detection.py         # Regex, merge, overlap, invalid model detection
|   +-- test_redaction.py         # Span replacement, overlap-safe ordering
|   +-- test_cache.py             # HMAC key safety, hit/miss, TTL, fallback
|   +-- test_scan_public.py       # Public endpoint: sizes, rates, errors, privacy
|   +-- test_scan_auth.py         # Auth endpoint: JWT, history ownership, cache-hit record
|   +-- test_gemini_provider.py   # Validation, error handling, mocked responses
|   +-- test_privacy_invariants.py# Sentinel: no raw text in DB or logs
|   +-- test_api_errors.py        # Error response schema consistency
|
+-- devpost/                      # Devpost learning workspace
|   +-- learner-profile.md
|   +-- scope.md
|   +-- prd.md
|   +-- spec.md
|
+-- .env.example
+-- .gitignore
+-- Dockerfile
+-- docker-compose.yml
+-- manage.py
+-- requirements.txt
+-- README.md
```

---

## External Services and Dependencies

### Google Gemini API

- **Endpoint:** managed by `google-generativeai` SDK internally
- **Auth:** `GEMINI_API_KEY` env var
- **Model:** `GEMINI_MODEL` env var (default `gemini-1.5-flash`)
- **Request format:** structured prompt + `response_mime_type="application/json"`
- **Rate limits (free tier):** ~15 RPM, 1M TPM, 1,500 req/day for Flash models — verify at https://ai.google.dev/gemini-api/docs/rate-limits
- **Cost:** free tier sufficient for demo
- **Timeout:** 20s (`GEMINI_TIMEOUT_SECONDS` env var)
- **Docs:** https://ai.google.dev/gemini-api/docs/

### Redis

- **Image:** `redis:7-alpine`
- **Connection:** `redis://redis:6379/0` (Docker Compose internal; `REDIS_URL` env var)
- **Client:** `redis-py` 5.x
- **Persistence:** none configured (ephemeral; data loss on restart is acceptable for cache)
- **Docs:** https://redis.io/docs/

---

## Docker Compose and Environment

### `docker-compose.yml`

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      redis:
        condition: service_healthy
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             gunicorn privacy_copilot.wsgi:application --bind 0.0.0.0:8000"

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

Two services only: `web` (Django + WhiteNoise + Gunicorn) and `redis`.

### `.env.example`

```dotenv
# Django
DJANGO_SECRET_KEY=replace-with-a-long-random-string
DEBUG=False

# Gemini
GEMINI_API_KEY=your-gemini-api-key-here
GEMINI_MODEL=gemini-1.5-flash
GEMINI_TIMEOUT_SECONDS=20

# Redis
REDIS_URL=redis://redis:6379/0

# Cache
CACHE_HMAC_SECRET=replace-with-a-separate-long-random-string
CACHE_TTL=86400
ANALYSIS_VERSION=v1

# Rate limits (optional overrides)
PUBLIC_SCAN_RATE=5/min
PUBLIC_MAX_TEXT_LENGTH=2000
AUTH_SCAN_RATE=30/min
AUTH_MAX_TEXT_LENGTH=10000
```

### `Dockerfile` (outline)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN python manage.py collectstatic --noinput
EXPOSE 8000
```

`collectstatic` copies `frontend/index.html` and `frontend/app.js` into `STATIC_ROOT` so WhiteNoise serves them. The catch-all URL rule returns `index.html` for `GET /`.

---

## Testing Strategy

Implements `prd.md > What We're Building` (completion checklist).

All tests use mocked Gemini responses so the suite runs without a live API key.

| Test file | Coverage |
|---|---|
| `test_detection.py` | Email regex, phone regex, merge, overlap resolution, invalid model detection dropped |
| `test_redaction.py` | Right-to-left span replacement, overlapping spans, invalid offset handling |
| `test_cache.py` | Key contains no raw-text substring; cache hit skips Gemini (mock call count = 1); TTL set correctly; bad cached schema falls back to fresh analysis; Redis GET error falls through; Redis SET error does not break scan |
| `test_scan_public.py` | 200 valid; 400 missing/empty text; 400 text > 2000 chars (no Gemini call); 429 on 6th request; 502 on invalid Gemini response; no ScanRecord created |
| `test_scan_auth.py` | 401 without token; 200 with valid token; 400 text > 10000 chars; history returns only own records; detail 404 for other user; cache-hit creates history with cache_hit=True |
| `test_gemini_provider.py` | JSONDecodeError raises ProviderResponseError; missing required field raises ProviderResponseError; invalid category dropped; API error raises ProviderResponseError |
| `test_privacy_invariants.py` | Sentinel text absent from all ScanRecord fields; sentinel absent from caplog output; no ScanRecord for public scans; cache value contains no sentinel |
| `test_api_errors.py` | Error schema shape correct for 400, 401, 429, 502, 500 |

```bash
# Local
pytest tests/ -v

# Docker
docker compose exec web pytest tests/ -v
```

---

## Important Failure Modes

- **Gemini unavailable / invalid JSON** → `ProviderResponseError` → HTTP 502 `PROVIDER_ERROR`. No fabricated result. Frontend: "Analysis unavailable — please try again shortly."

- **Redis connection refused** → `CacheService.get()` catches `RedisError`, logs non-sensitive warning, returns `None`. Scan continues. `CacheService.set()` also catches and ignores. No HTTP 500 for cache failure alone.

- **`CACHE_HMAC_SECRET` missing at startup** → `AppConfig.ready()` raises `ImproperlyConfigured`. Container fails to start with a clear error message. Operator must supply the secret.

- **JWT expired** → API returns 401. Frontend clears `sessionStorage` token and prompts re-login without losing textarea content.

- **`gemini-1.5-flash` model unavailable** → `GeminiProvider` raises `ProviderResponseError`. HTTP 502. Fix: change `GEMINI_MODEL` env var to a supported model, restart.

---

## What Was Simplified and Why

- **SQLite instead of PostgreSQL** — zero extra infrastructure. Schema uses only portable ORM constructs; switching later requires changing `DATABASES['default']` only.
- **WhiteNoise instead of Nginx** — eliminates a third Docker service. Sufficient for demo throughput.
- **Manual `analysis_version` bump** — string constant `"v1"` in settings; bumped when detection logic changes incompatibly. Auto-derivation adds complexity with no POC benefit.
- **NFC + whitespace-collapse normalization** — simple, deterministic, does not alter semantic content. Prevents spurious cache misses from encoding inconsistencies.
- **No user registration UI** — test users created via `manage.py createsuperuser`. Registration is out of scope for the POC.
- **Single Gemini provider built** — `PrivacyAnalysisProvider` ABC is in place; OpenAI adapter requires only a new file implementing the interface.

---

## Decisions and Open Issues

### Learner-confirmed decisions

| Decision | Choice | Reason / Tradeoff |
|---|---|---|
| Frontend serving | Django + WhiteNoise | Eliminates third container; simpler solo demo |
| Gemini model | `gemini-1.5-flash` (env-configurable) | Fast Flash tier; switch by env var only |
| Cache-key construction | HMAC-SHA256 keyed digest | Keyed construction prevents raw-text exposure in cache storage |
| HTTP 404 for unauthorized history records | 404 (not 403) | Avoids confirming existence of another user's record |
| Token storage | `sessionStorage` | Reduces persistent XSS surface; acceptable for demo |
| `analysis_version` management | Manual string constant `"v1"` | Simple; sufficient for POC |
| Text normalization | NFC + whitespace collapse | Deterministic; does not alter text meaning |

### Genuine uncertainty discussed

**Why HMAC-keyed keys differ from plain hashes:** A plain `SHA256(text)` is unkeyed — anyone who knows the algorithm can precompute it from any text and observe whether a given text was scanned (a timing/oracle attack on Redis key names). A `HMAC-SHA256(secret, text)` requires the secret. For this application the main practical benefit is that the cache key cannot be reverse-engineered to confirm whether a specific sensitive string was submitted. Implementation: `hmac.new(secret, payload, hashlib.sha256)` in `CacheService.make_key()`.

### Resolved open questions (from `prd.md`)

| # | Resolution |
|---|---|
| OQ-1 | `GEMINI_MODEL` is env-configurable; verify on first integration test; change env var if needed |
| OQ-2 | Django + WhiteNoise (locked decision B) |
| OQ-3 | Manual string constant `"v1"` in `settings.py` |
| OQ-4 | NFC normalization + whitespace collapse in `CacheService.make_key()` |

### Remaining action item

> **During `5-build` step 1 (Gemini integration):** verify that `gemini-1.5-flash` is available under the configured API key. If not, update `GEMINI_MODEL` in `.env` and document the chosen model identifier in `spec.md`. No other code changes required.
