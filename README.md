BUILD_INSTRUCTIONS.md

Audience: an AI coding agent.
Goal: build a Laravel-style monolith in Python using FastAPI + SQLAlchemy that serves server-rendered pages and JSON APIs, with a strict config policy and a Laravel-like folder layout.

⸻

0) Non-negotiable constraints
	•	Keep the static web root as public/ (served at /static).
	•	Only files in config/ may read .env.
All other code must use a global config registry via config.get() / config.set().
	•	Provide both web (Jinja) and API (JSON) routes.
	•	Ship with SQLAlchemy 2.x (async) + Alembic and an initial users table.
	•	Include a minimal Users CRUD slice (service + endpoints).
	•	Include auto API docs and a small homepage.
	•	Include a simple “artisan-like” CLI using Typer for scaffolding (make:model/service/controller/migration).

⸻

1) Tech stack (fixed)
	•	FastAPI (Starlette), Uvicorn (dev) / Gunicorn+Uvicorn workers (prod)
	•	SQLAlchemy 2.x (async), Alembic
	•	Pydantic v2 + pydantic-settings
	•	Jinja2 templates
	•	Passlib[bcrypt], PyJWT (auth ready; basic utilities ok)
	•	Redis & Celery placeholders (optional hooks)
	•	Structlog/loguru, OpenTelemetry, Sentry (placeholders; optional)
	•	Pytest, pytest-asyncio, httpx (tests)

⸻

2) Project layout (MUST match)

your_app/
├─ app/
│  ├─ Http/
│  │  ├─ Controllers/
│  │  └─ Requests/
│  ├─ Models/
│  ├─ Policies/
│  ├─ Services/
│  ├─ Providers/
│  └─ Jobs/
├─ bootstrap/
│  └─ app.py
├─ config/
│  ├─ __init__.py
│  ├─ registry.py
│  ├─ app.py
│  ├─ database.py
│  ├─ auth.py
│  └─ storage.py
├─ database/
│  ├─ migrations/
│  │  ├─ env.py
│  │  └─ versions/
│  └─ seeders/
├─ routes/
│  ├─ web.py
│  └─ api.py
├─ resources/
│  ├─ views/
│  └─ lang/
├─ public/
├─ storage/
├─ tests/
├─ docs/
├─ artisan.py
├─ app.py
├─ alembic.ini
├─ pyproject.toml
└─ README.md


⸻

3) Step-by-step build plan

Step 1 — Initialize project & dependencies
	1.	Create the directory tree above.
	2.	Create pyproject.toml with dependencies:

fastapi
uvicorn[standard]
jinja2
orjson
pydantic-settings
python-multipart
sqlalchemy[asyncio]>=2.0
alembic
asyncpg
passlib[bcrypt]
pyjwt
redis
slowapi
boto3
aiofiles
typer[all]
pytest
pytest-asyncio
httpx


	3.	Create .env.example with:

APP_NAME=Your App
APP_DEBUG=true
APP_ENV=local
DB_URL=sqlite+aiosqlite:///./dev.db
JWT_SECRET=dev-secret
JWT_ALGO=HS256
JWT_TTL_MIN=30



Step 2 — Config registry (the only .env readers live in config/)
	•	config/registry.py

_STORE = {}
def set(path, value): _STORE[path] = value
def get(path, default=None): return _STORE.get(path, default)
def all(): return dict(_STORE)


	•	config/__init__.py re-exports get, set, all.
	•	config/app.py, config/database.py, config/auth.py, config/storage.py use pydantic-settings to read .env and publish to registry with set(...).

Step 3 — App factory & entrypoint
	•	bootstrap/app.py:
	•	Load all config loaders (cfg_app.load(), cfg_db.load(), cfg_auth.load(), cfg_storage.load()).
	•	Create FastAPI(title, version).
	•	Mount public/ at /static.
	•	Add CORS + GZip middleware.
	•	Include routers from routes/web.py and routes/api.py.
	•	app.py:

from bootstrap.app import create_app
app = create_app()



Step 4 — Database provider & models
	•	app/Providers/db.py: create async engine using config.get("database.url") and async_sessionmaker.
	•	app/Models/base.py:
	•	SQLAlchemy Base + TimestampsMixin (created_at/updated_at with func.now()).
	•	app/Models/user.py: User model with id, name, email (unique), password_hash + timestamps.
	•	Wire Alembic:
	•	alembic.ini with sqlalchemy.url (sqlite dev default).
	•	database/migrations/env.py imports Base and models; sets target_metadata=Base.metadata.
	•	Create initial migration in database/migrations/versions/ for users table.

Step 5 — Web & API routes
	•	routes/web.py: Jinja home.html (in resources/views/) and render with context app_name=config.get("app.name").
	•	routes/api.py:
	•	APIRouter() for GET /api/v1/health.
	•	Include Users endpoints (GET /api/v1/users, POST /api/v1/users).
	•	app/Http/Requests/user.py: UserCreate (name, email, password) and UserOut (id, name, email).

Step 6 — Services
	•	app/Services/users.py:
	•	UserService with methods list() and create() (hash password with passlib).

Step 7 — CLI (artisan-style)
	•	artisan.py (Typer):
	•	make:model <Name>
	•	make:service <Name>Service --resource <Name>
	•	make:controller <Name>Controller --resource <Name>
	•	make:migration <name>
	•	Generators must:
	•	Place files in correct folders.
	•	Add class and basic stubs.
	•	(Optional) Add comment anchors:
	•	Models: # MODEL_START / # MODEL_END
	•	Services: # SERVICE_START / # SERVICE_END
	•	Routes: # ROUTES_START / # ROUTES_END

Step 8 — Templates & public assets
	•	resources/views/home.html: simple landing page referencing /static/*.
	•	public/robots.txt, public/favicon.ico (stub).

Step 9 — Tests
	•	tests/test_health.py:
	•	Start app via from app import app.
	•	Use httpx.AsyncClient to GET /api/v1/health and assert {"status":"ok"}.
	•	(Optional) test users list/create.

Step 10 — Docs
	•	README.md:
	•	Quick start, run commands, API docs URL, migrations usage.
	•	docs/ (optional MkDocs) and export openapi.json later.

⸻

4) Commands to run (execute in order)

Install & run (dev)

# (venv recommended)
pip install -e .
uvicorn app:app --reload

Database

alembic upgrade head

Try the API

curl http://127.0.0.1:8000/api/v1/health
curl http://127.0.0.1:8000/api/v1/users
curl -X POST http://127.0.0.1:8000/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Aliya","email":"aliya@example.com","password":"secret"}'

Scaffolding examples

python artisan.py make:model Product
python artisan.py make:service ProductService --resource Product
python artisan.py make:controller ProductController --resource Product
python artisan.py make:migration create_products_table


⸻

5) Coding standards & guardrails
	•	Config access: Only via from config import get. Do not call os.getenv outside config/.
	•	Requests/Responses: Validate with Pydantic models; return typed responses; prefer consistent error shapes (Problem+JSON optional).
	•	Routing: Organize by feature; use prefixes (/api/v1/...) and tags.
	•	DB: Keep models thin; put logic in Services; always commit/refresh correctly.
	•	Security: Use passlib for passwords; CSRF only for server-rendered forms (web). Set CORS carefully in prod.
	•	Static assets: Served from public/ at /static.
	•	Tests: For each route added, provide at least one test.

⸻

6) Acceptance checklist (must pass)
	•	public/ is mounted at /static and used by the homepage.
	•	.env is only read in config/*; global registry is used everywhere else.
	•	GET /api/v1/health returns {"status":"ok"}.
	•	Users model + migration exists; alembic upgrade head succeeds.
	•	POST /api/v1/users creates a user with a hashed password; GET /api/v1/users lists users.
	•	CLI artisan.py provides make:model, make:service, make:controller, make:migration.
	•	pytest passes for the health test (and any additional tests added).
	•	README documents run steps and endpoints.

⸻

7) Optional next milestones
	•	Add JWT login/refresh + guards (session or JWT strategy).
	•	Add Redis cache & slowapi rate-limits.
	•	Add Celery + Beat with example jobs.
	•	Add SQLAdmin or a small Jinja admin area.
	•	Add OpenTelemetry traces + Sentry error reporting.
	•	Multi-tenancy by subdomain with per-tenant DB sessions.

⸻

8) Handover notes for AI agent
	•	Be deterministic: follow the folder names and file names exactly.
	•	Be conservative: don’t invent settings or read env outside config/*.
	•	Keep diffs small: add minimal working code for each step before moving on.
	•	Run tests frequently: after creating health route and users endpoints.
	•	Stop on errors: if a command fails, fix it before continuing.

⸻

End of file.
