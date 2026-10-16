---
doc: scope
status: approved
---

# Privacy Copilot for AI Workflows

A privacy checkpoint that detects sensitive information in text, explains the risks in plain English, and returns a deterministic redacted version — before a user pastes anything into an external AI tool.

## The Unique Kernel

The product is not a PII finder. It is a **privacy-conscious decision aid**: it combines local deterministic detection with contextual Gemini analysis, explains *why* the text poses a risk, and produces a redacted preview — while being honest that detection is imperfect, that text sent to Gemini leaves the application, and that redaction is not a legal compliance guarantee. That combination of real-world honesty, hybrid detection, and HMAC-keyed privacy-conscious caching is what makes this distinctive.

## Who It's For

A developer debugging an application error. They have a stack trace or API response open, they need to understand it, and their first instinct is to paste it into ChatGPT or Gemini. They don't stop to check whether it contains a customer email, an internal system path, or an API key. This tool is the pause between that instinct and that paste.

Same loop, slightly different stakes: a business analyst with a customer-feedback export, or a legal consultant with case notes. In every case the user is not trying to commit a privacy violation — they just don't have a fast way to check before they share.

## The Core Loop

1. User opens the web interface.
2. User pastes text into the input area and submits.
3. Backend runs deterministic regex detection locally, then sends the minimum necessary context to Gemini for contextual analysis.
4. Backend validates the model response, merges findings, and runs deterministic redaction.
5. User sees: a plain-English risk summary, detected entity categories, and a redacted preview they can copy.
6. If they've logged in, the scan metadata and sanitized output are saved to their history.
7. If the same text is scanned again (within the TTL), the cached result is returned without a new LLM call.

## Inspiration & Identity

The product should feel like a calm, professional tool — not a warning siren. Useful, clear, and honest. Reference feel: a linter or a grammar checker that surfaces issues and explains them rather than blocking the user. The interface is secondary to the backend quality; clean and functional over decorative.

## Why This Matters to the Learner

"I want to genuinely master the plan-first, spec-driven development workflow with an AI coding agent." This project is the real application of that: vision → scope → PRD → spec → incremental build. The privacy-first Redis caching, provider abstraction, and testing of sensitive code paths are the technical depth that makes it worth doing properly.

## What "Working" Looks Like

A reviewer opens the Docker Compose stack, pastes a short paragraph containing a name, email, and phone number into the public scan form, and sees a risk explanation and a redacted version in the browser. They log in, run the same scan again, and the response is immediate — a cache hit. They open their scan history and see only their own records. They stop the Redis container, run another scan, and the application continues working. That sequence, on screen, in under two minutes, is the demo.

## The POC Boundary

**In:**
- `POST /api/v1/scans/public/` — unauthenticated scanning with rate limits and input-size restrictions; no history saved.
- `POST /api/v1/scans/` — authenticated scanning; scan metadata and sanitized output persisted per user via Django ORM + SQLite.
- `GET /api/v1/scans/history/` — authenticated user's own records only.
- JWT authentication: `POST /api/token/`, `POST /api/token/refresh/`.
- Hybrid detection: deterministic regex (email, phone, common patterns) + Gemini contextual analysis.
- Deterministic redaction with `[EMAIL]`, `[PHONE]`, `[NAME]`, `[ADDRESS]` labels; overlap-safe.
- Gemini provider via a clean abstraction; structured JSON output with server-side validation.
- Privacy-conscious Redis caching: HMAC-SHA256 keyed cache keys, 24h configurable TTL, eligible-result-only caching, graceful Redis fallback.
- Lightweight frontend: HTML5, vanilla JS, Tailwind CSS via CDN; scan form, risk summary, redacted preview with copy, login, scan history.
- Docker Compose: three services — frontend (static), backend (Django), Redis.
- Unit and API tests: detection, redaction, auth, authorization, cache hits/misses, Redis failure, no-raw-text persistence guarantee.
- `.env.example` with placeholder values only; actual secrets gitignored.

**Privacy invariants (non-negotiable in every layer):**
1. Raw submitted text never persisted in SQLite, Redis, or logs.
2. Gemini transmission minimized and accurately disclosed.
3. Model findings validated before influencing redaction.
4. Cache hits never bypass auth or authorization.
5. Redis failures never leak raw input through logs or error messages.
6. Application never claims perfect detection, guaranteed anonymization, or legal compliance.

## Later

- OpenAI adapter (the abstraction is designed for it; implementation deferred).
- PostgreSQL migration (schema designed to support it from day one; SQLite for the demo).
- More detection categories: credentials/API keys, IBANs, national ID formats.
- User-managed cache invalidation (clear my cached scans).
- Admin analytics dashboard.
- Per-user rate-limit configuration.

## Explicitly Cut

- **React / Vue / Next.js frontend** — adds build complexity with no benefit to proving the backend privacy workflow. HTML + Tailwind via CDN is sufficient for the demo.
- **Second fully implemented AI provider (OpenAI)** — the abstraction will support it; the implementation will not. Two providers to maintain doubles testing surface for no additional proof-of-concept value.
- **Multiple document formats (PDF, DOCX, CSV)** — file parsing is a separate engineering problem unrelated to the core privacy loop. Text input only.
- **Enterprise access controls (teams, roles, org-level history)** — out of scope for a single-user hackathon proof of concept.
- **Production-scale deployment (cloud hosting, managed Redis, TLS termination)** — Docker Compose for local demo only.
- **Guaranteed anonymization or legal compliance claims** — explicitly excluded as a product promise, not just a build task. The application must say so.
