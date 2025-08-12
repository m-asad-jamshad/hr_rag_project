# Codefinity HR Bot — RAG Application

A Retrieval-Augmented Generation (RAG) web app built with **Django** that indexes HR documents and answers employee HR questions using **OpenAI** (LLM + embeddings) and a vector store (Chroma). The front-end uses Bootstrap with a glassmorphic theme. The app supports user signup/login, PDF uploads, ingestion to a vector store, and a chat UI.

> ⚠️ **Security note:** Do **not** commit secrets. Rotate any exposed API keys immediately.

---

## Table of contents

- [Features](#features)  
- [Prerequisites](#prerequisites)  
- [Quickstart — Local (no Docker)](#quickstart---local-no-docker)  
- [Quickstart — Docker Compose](#quickstart---docker-compose)  
- [Environment variables (.env)](#environment-variables-env)  
- [Manage the database & migrations](#manage-the-database--migrations)  
- [Key project URLs / API endpoints](#key-project-urls--api-endpoints)  
- [Common errors & fixes (you encountered these)](#common-errors--fixes-you-encountered-these)  
- [Production recommendations](#production-recommendations)  
- [Where to look next / development notes](#where-to-look-next--development-notes)  
- [License & contact](#license--contact)

---

## Features

- Signup / Login (custom user model with email as identifier)
- Upload PDF documents (HR policies) and persist them
- Ingest PDFs into a Chroma vector store using OpenAI embeddings (with a possible local HF fallback)
- Chat UI (AJAX) that queries the vector store and returns answers & sources
- Glassmorphic Bootstrap-based theme
- Defensive handling for OpenAI rate limits and graceful error messaging

---

## Prerequisites

- Python 3.10+ (3.11 recommended)
- PostgreSQL (local or Docker)
- OpenAI API key (or configure local embeddings if you install HF & sentence-transformers)
- (Optional) Docker & Docker Compose for containerized run

---

## Quickstart — Local (no Docker)

1. Clone repo and create virtualenv:
   ```bash
   git clone <your-repo-url>
   cd hr_rag_project
   python -m venv .venv
   source .venv/bin/activate    # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```

2. Create `.env` from template:
   ```bash
   cp .env.example .env
   # edit .env, set DJANGO_SECRET_KEY, OPENAI_API_KEY, POSTGRES_*
   ```

3. Configure Postgres connection details in `.env` (or create local DB). Example `.env` keys are below.

4. Run migrations and create superuser:
   ```bash
   python manage.py migrate
   python manage.py createsuperuser
   ```

5. Start development server:
   ```bash
   python manage.py runserver
   ```

6. Visit `http://127.0.0.1:8000` — Home, Signup, Login, Upload, Chat.

---

## Quickstart — Docker Compose

1. Copy `.env.example` → `.env` and fill in secrets (do not commit `.env`).

2. Build & run:
   ```bash
   docker compose up --build
   ```

3. App will be at `http://127.0.0.1:8000`. Postgres container is named `db` by default.

---

## Environment variables (`.env`)

Use a local `.env` (never commit). Minimal variables the app expects:

```env
# Django
DEBUG=True
DJANGO_SECRET_KEY=replace-with-your-secret

# OpenAI
OPENAI_API_KEY=sk-xxxx

# Postgres (match these names to settings.py)
POSTGRES_DB=hr_rag_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=supersecret
POSTGRES_HOST=localhost   # 'db' when using docker compose
POSTGRES_PORT=5432

# Vector store
CHROMA_DIR=chroma_db
USE_CHROMA_LOCAL=false
```

---

## Manage the database & migrations

- Create / apply migrations:
  ```bash
  python manage.py makemigrations
  python manage.py migrate
  ```

- Create admin user:
  ```bash
  python manage.py createsuperuser
  ```

---

## Key project URLs / API endpoints

- `/` — Home page
- `/signup/` — Signup page
- `/login/` — Login page
- `/logout/` — Logout
- `/upload/` — Upload PDF(s)
- `/chat/` — Chat UI (protected — requires login)
- `/chat/query/` — (optional) older synchronous endpoint
- `/chat/api/` — AJAX endpoint used by `static/rag_app/chat.js` (POST; form field `question`)

> Make sure your `urls.py` exposes `/chat/api/` if your front-end `chat.js` calls it. If you use `/chat/query/`, align JS to call that URL.

---

## Common errors & fixes (you ran into these)

Here are the precise causes and durable fixes for the errors you’ve seen:

### 1. `RateLimitError / insufficient_quota` (OpenAI 429)
**Cause:** Your OpenAI account exceeded quota or key invalid.  
**Fix:**
- Rotate/replace OpenAI API key and ensure `OPENAI_API_KEY` is set.
- Add try/except around ingestion and fallback to local HF embeddings or queue ingestion for later.
- Avoid blocking web requests for ingestion — use background tasks (Celery/RQ).

### 2. `405 Method Not Allowed` at `/chat/`
**Cause:** Frontend fetch used wrong HTTP method or wrong endpoint (GET vs POST) or CSRF mismatch.  
**Fix:**
- Confirm `fetch()` uses correct method and target URL. For submitting a question via AJAX, use POST.
- If you have a view decorator `@require_http_methods(["GET","POST"])`, ensure the request method matches.

### 3. `ValueError: Cannot query "user@example.com": Must be "CustomUser" instance.`
**Cause:** You passed a string or `SimpleLazyObject` to a model foreign key filter that expects a user instance.  
**Fix:**
- Use `ChatHistory.objects.filter(user=request.user)` (gives a CustomUser) or store `user_id` integer: `filter(user_id=request.user.id)`.
- Defensive code: resolve by id when older records used strings.

### 4. `TemplateDoesNotExist: chat.html` or other templates
**Cause:** Template file not found in configured template dirs or wrong path in `render()`.  
**Fix:**
- Ensure `templates/` or `app/templates/rag_app/chat.html` exists and `TEMPLATES['DIRS']` in settings points to your templates folder.
- Use `render(request, 'rag_app/chat.html', ...)` when template is inside `rag_app/templates/rag_app/`.

### 5. Frontend says `server returned HTML (check server log or network tab)`
**Cause:** The AJAX call expected JSON but received an HTML redirect or error page (often because user is unauthenticated and Django redirected to login HTML).  
**Fix:**
- AJAX should include `X-Requested-With: XMLHttpRequest` and `Accept: application/json` headers. The backend should detect AJAX and return `JsonResponse({'error':'not_authenticated'}, status=401)` rather than redirect.
- Ensure CSRF token is set in headers (`X-CSRFToken`).

### 6. `TypeError: Failed to fetch`
**Cause possibilities:**
- Network blocked (CORS), server unreachable, invalid URL, blocked by browser due to mixed content (HTTP/HTTPS), or the fetch rejected due to no CORS allowed.
**Fix:**
- For same-origin requests in dev, ensure you call the correct URL (`/chat/api/`) and server is running.
- Open browser DevTools → Network tab: see request details and response.
- If using cross-origin, configure CORS headers (e.g., django-cors-headers).

### 7. `NameError: name 'openai' is not defined`
**Cause:** Code attempted `openai.api_key = ...` without importing `openai`.  
**Fix:** `import openai` at top of views or remove direct openai usage and rely on langchain/openai packages which read env var.

---

## Production recommendations

- **Set `DEBUG = False`** and configure `ALLOWED_HOSTS`.
- **Rotate any exposed keys immediately.**
- Serve static files via CDN or a web server (Nginx), not Django `runserver`.
- Use background workers for ingestion (Celery/RQ). Ingestion may be long-running and should not block request-response.
- Use a secrets manager (Vault, AWS Secrets Manager, or environment variables in the host/containers).
- Use TLS for all production traffic.
- If you don’t intend to run local HF models, remove heavy ML packages from `requirements.txt` to make builds smaller.

---

## Where to look next / development notes

- `templates/rag_app/` — edit the HTML and glassmorphic CSS.
- `static/rag_app/chat.js` — frontend AJAX logic (CSRF and headers).
- `rag_app/views.py` — views for signup/login/upload/chat/chat_api.
- `rag_app/vector_store.py` — ingestion and Chroma usage; it may include local HF fallback logic.
- Check `hr_rag/settings.py` for env var names and ensure `.env` matches.
- Consider adding tests for views and vector-store ingestion logic.

---

## Helpful debugging steps

1. Reproduce the failing request with `curl` to inspect response:
   ```bash
   curl -v -X POST http://127.0.0.1:8000/chat/api/ \
       -H "X-CSRFToken: <token>" \
       -F "question=hello"
   ```
2. Use browser DevTools Network tab to view actual request/response headers and body.
3. Check Django runserver console logs for tracebacks.
4. For missing template errors, print `settings.TEMPLATES` and verify the filesystem path.

---

## Example useful commands

- Run dev server:
  ```bash
  python manage.py runserver
  ```
- Run migrations:
  ```bash
  python manage.py migrate
  ```
- Create superuser:
  ```bash
  python manage.py createsuperuser
  ```
- Collect static (for production):
  ```bash
  python manage.py collectstatic --noinput
  ```

---

## License & contact

This repository is for internal Codefinity Solutions use. If you want me to produce optimized Dockerfile, slim `requirements-prod.txt`, or a Celery + Redis example for background ingestion, tell me which direction you prefer and I’ll provide code.

