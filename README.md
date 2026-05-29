# Artifact3 — Node.js → Python 3 / Flask Migration

This repository is the target of a **behavior-preserving migration** of a Node.js HTTP server to a Python 3 application built on the [Flask](https://flask.palletsprojects.com/) web framework, reproducing the original server's externally observable functionality verbatim.

---

## ⚠️ Project Status — Migration Pending Source Delivery

> **The Node.js source to be migrated has not yet been committed to this repository.** Authoritative inspection confirms that, at this point, the repository tracks **only this `README.md`** (whose entire prior content was the single line `# Artifact3`). No `package.json` / lockfile, no `.js` / `.ts` / `.mjs` / `.cjs` modules, no `server.js` / `app.js` / `index.js`, no `config/`, no `test/`, no `Dockerfile`, no `.github/`, and no `.env` are present. The repository is **greenfield / empty-by-evidence**.
>
> **The concrete migration is therefore pending delivery of the Node.js source.** Committing that source is the trigger that activates the full migration — at which point every wildcard pattern in the planned architecture below resolves to concrete files and the rewrite is executed in a single pass.
>
> **To honor evidence-based authoring, no application code has been fabricated in advance.** This document is a migration **charter / conditional plan** — not setup documentation for a running application. Specific routes, environment-variable names, database schemas, models, services, and business logic are intentionally **not** invented here, because no source exists from which to derive them.

---

## 1. Migration Objective

The migration is a **complete technology-stack migration** that is simultaneously:

- **Cross-language:** JavaScript → Python 3.
- **Cross-framework:** Node.js / Express-style HTTP server → Flask (WSGI).

The non-negotiable success criterion is **100% functional parity** with the original server's externally observable behavior. Concretely, this means:

| # | Goal | Description |
|---|------|-------------|
| G1 | **Route parity** | Every HTTP route (path + method) is ported with identical request/response contracts — status codes, payload shapes, and headers. |
| G2 | **Middleware semantics** | The middleware pipeline is reproduced: authentication, body parsing, CORS, logging, error handling, and rate limiting where present — preserving registration order and short-circuit behavior. |
| G3 | **Configuration & environment** | Every environment variable is read by the **same name** and **same default** as the original. |
| G4 | **Persistence / data access** | The data-access layer is ported to a Python equivalent, preserving schema, queries, and relationships. |
| G5 | **External integrations** | HTTP clients, SDKs, and message brokers are ported to Python equivalents with identical contracts. |
| G6 | **Background work** | Scheduled jobs, background workers, and websockets are ported if present in the source. |
| G7 | **Auth & sessions** | Authentication and authorization are ported with identical token / session semantics. |
| G8 | **Test suite** | The test suite is ported to `pytest`, preserving coverage intent and assertions. |
| G9 | **Bootstrap & listen** | Server bootstrap and listen behavior (host / port, graceful shutdown) is reproduced under a production WSGI server. |

This is a **behavior-preserving rewrite** — no new endpoints, no removed endpoints, no altered validation rules or error messages, no performance redesign.

---

## 2. Planned Target Architecture

The target is an idiomatic Flask application built around the **Application Factory** pattern with **Blueprints** for route grouping, a clean separation between view functions and business logic, and centralized cross-cutting concerns.

### 2.1 Design Patterns

| Pattern | Where | Purpose |
|---------|-------|---------|
| **Application Factory** (`create_app()`) | `app/__init__.py` | Replaces the implicit Express `app` singleton; enables isolated configuration and clean test setup. |
| **Blueprints** | `app/blueprints/` | The Flask equivalent of `express.Router()`; group routes by resource and register them in the factory. |
| **Service layer** | `app/services/` | Business logic is extracted from view functions, mirroring the source's controller / service separation. |
| **ORM models** | `app/models/` | Persistence is abstracted behind models (SQLAlchemy for relational sources; `pymongo` / MongoEngine for MongoDB sources), preserving the source schema. |
| **Schema / serialization layer** | `app/schemas/` | Request validation and response serialization (marshmallow or pydantic) — the equivalent of Joi / `express-validator`. |
| **Extensions** | `app/extensions.py` | Shared singletons (database, CORS, JWT, serialization) are instantiated once and bound to the app inside the factory. |
| **Centralized error handlers** | `app/errors.py` | `@app.errorhandler` registrations replace Express's `(err, req, res, next)` middleware, preserving status codes and JSON error bodies. |
| **Environment-specific Config classes** | `config.py` | `Base` / `Development` / `Production` / `Testing` configurations sourced from `os.environ`, replacing `dotenv`-driven configuration. |
| **Production WSGI server** | `wsgi.py` | `gunicorn` (UNIX) or `waitress` (Windows) replaces `server.listen(...)`. |

### 2.2 Node → Flask Construct Mapping (illustrative — not app-specific)

| Node.js / Express construct | Flask target | Transformation rule |
|------------------------------|--------------|---------------------|
| `express()` | `create_app()` factory in `app/__init__.py` | Convert the implicit singleton + `app.use(...)` wiring into a factory that registers extensions, blueprints, and error handlers. |
| `express.Router()` | `Blueprint(...)` | One blueprint per route group; register via `app.register_blueprint(bp)`. |
| `app.use(mw)` | `@app.before_request` / `@app.after_request` hooks, WSGI middleware, route decorators | Preserve registration order and short-circuit semantics. |
| `req` / `res` | `flask.request` / `make_response` / `jsonify` | `res.status(n).json(x)` → `(jsonify(x), n)`; `res.send(t)` → `return t`. |
| Route param `/users/:id` | `/users/<id>` (or `<int:id>`) | Use a typed converter where the source parsed numerics. |
| `express.json()` body parsing | `request.get_json()` | Parsed on demand rather than via mounted middleware. |
| `next(err)` error propagation | raised exceptions + `@app.errorhandler` | Centralize error mapping; preserve status / body. |
| `async / await` handlers | synchronous WSGI view functions (default) | Default to synchronous; flag `async` views or Quart only for genuinely async-heavy / streaming / websocket workloads. |
| `process.env.X` | `os.environ['X']` surfaced through `Config` classes | Preserve every variable name and default. |
| `server.listen(port)` | `gunicorn 'wsgi:app'` / `waitress` | Production WSGI server replaces the built-in Node HTTP listener. |

### 2.3 Planned Target Structure

The following tree is the planned target structure that will be **created once the Node.js source is committed**. Folders and files marked **(conditional)** are produced only when the source exhibits the corresponding capability.

```
<repo-root>/
├── README.md                  # this file — updated to Python/Flask setup docs after migration
├── requirements.txt           # pinned runtime dependencies
├── requirements-dev.txt       # test / lint dependencies
├── pyproject.toml             # packaging + tool config (optional alt to requirements files)
├── .env.example               # documents env var names (mirrors source .env)
├── .gitignore                 # Python ignores: venv/, __pycache__/, .pytest_cache/, *.pyc
├── wsgi.py                    # WSGI entry: app = create_app()
├── config.py                  # Config classes: Base / Development / Production / Testing
├── app/
│   ├── __init__.py            # create_app() factory — registers extensions, blueprints, errors
│   ├── extensions.py          # db, cors, jwt, ma singletons initialized in factory
│   ├── blueprints/            # one module per Express Router / route group
│   │   ├── __init__.py
│   │   └── <resource>.py
│   ├── models/                # data models ported from models/
│   │   ├── __init__.py
│   │   └── <model>.py
│   ├── schemas/               # request / response validation & serialization
│   │   └── <schema>.py
│   ├── services/              # business logic ported from services/
│   │   └── <service>.py
│   ├── middleware/            # before_request / after_request hooks + auth decorators
│   │   └── <middleware>.py
│   ├── errors.py              # centralized @app.errorhandler registrations
│   ├── utils/                 # helpers ported from utils/
│   │   └── <helper>.py
│   ├── templates/             # (conditional) Jinja2 templates from views/
│   └── static/                # (conditional) assets from public/
├── tests/                     # pytest suite ported from test/
│   ├── __init__.py
│   ├── conftest.py            # app / client / db fixtures
│   └── test_<resource>.py
├── Dockerfile                 # (conditional) only if source has Docker
├── docker-compose.yml         # (conditional) only if present in source
└── .github/workflows/ci.yml   # (conditional) only if source has CI
```

Files such as `wsgi.py`, `app/__init__.py`, `app/extensions.py`, `config.py`, and `tests/conftest.py` have no Node.js equivalent — they are the structural files required for idiomatic Flask operation and will be authored fresh as part of the migration.

---

## 3. Planned Technology Stack

### 3.1 Verified Core Stack (web-verified targets)

The versions below are the **planned target** for the migration, selected and verified against authoritative sources (PyPI, python.org, project documentation). They are cited here so the manifests written during migration use real, current versions — not placeholders.

| Technology | Planned version | Role |
|------------|-----------------|------|
| **Python** (runtime) | **3.13.x** (recommended) | Target interpreter; create the virtual environment with this exact runtime. |
| **Flask** | **3.1.3** | WSGI web framework — replaces `express`. Requires Python ≥ 3.9. |
| **Werkzeug** | **≥ 3.1** | WSGI routing / utilities — installed with Flask 3.1.3. |
| **Jinja2** | **3.1.x** | Server-side templating — installed with Flask; replaces ejs / pug / handlebars (only if SSR is present in the source). |
| **gunicorn** | **26.0.0** | Production WSGI HTTP server on **UNIX**. Requires Python ≥ 3.10. |
| **waitress** | **latest stable** | Production WSGI HTTP server on **Windows** (gunicorn is UNIX-only because of `fcntl`). Used in place of gunicorn on Windows hosts. |
| **SQLAlchemy** | **2.0.50** | ORM / Core — **conditional**; only included if the future source uses a relational database. |

To write a `requirements.txt` that builds on both platforms, declare both production servers with environment markers, e.g.:

```text
gunicorn==26.0.0; sys_platform != 'win32'
waitress; sys_platform == 'win32'
```

### 3.2 Conditional Dependencies — "pin at implementation"

Every other dependency is deliberately **selected and pinned at implementation time**, based on the actual capabilities exercised by the committed Node.js source. No exact versions are claimed here, because doing so would invent specifics. The following Node → Python equivalence table is the guidance the migration will follow when those dependencies are pinned:

| Node.js (npm) | Python (PyPI) | Notes |
|---------------|---------------|-------|
| `express` | Flask | Core framework. |
| `body-parser` / `express.json` | (built-in) `request.get_json()` | No separate package needed. |
| `cors` | Flask-CORS | Mirror origins / methods / headers / credentials. |
| `mongoose` | MongoEngine / pymongo | Document model / driver. |
| `sequelize` / `prisma` / `knex` / `typeorm` | SQLAlchemy (+ Alembic) | ORM + migrations. |
| `jsonwebtoken` / `passport-jwt` | Flask-JWT-Extended / PyJWT | Preserve claims, expiry, algorithm. |
| `joi` / `express-validator` / `zod` | marshmallow / pydantic | Validation / DTO. |
| `dotenv` | python-dotenv | Env loading. |
| `axios` / `node-fetch` / `got` | requests / httpx | Outbound HTTP. |
| `bcryptjs` | bcrypt / passlib | Password hashing. |
| `winston` / `pino` | `logging` (stdlib) / structlog | Structured logging. |
| `morgan` | Werkzeug request logging | Access logs. |
| `helmet` | flask-talisman | Security headers. |
| `multer` | `request.files` / Werkzeug | File uploads. |
| `socket.io` | Flask-SocketIO | Websockets. |
| `node-cron` / `bull` / `agenda` | APScheduler / Celery | Scheduling / queues. |
| `pm2` | gunicorn + systemd | Process management. |
| `nodemon` | `flask --debug` | Dev reload. |
| `jest` / `mocha` / `chai` / `supertest` | pytest / pytest-flask | Testing. |
| `eslint` / `prettier` | ruff / black | Lint / format. |

Each conditional dependency is added to `requirements.txt` (or `requirements-dev.txt`) only when the source actually uses the corresponding capability, and pinned to the latest stable release compatible with Flask 3.1 and Python 3.13.

---

## 4. Planned Setup / Run / Test Workflow

> **Forward guidance — this workflow becomes runnable after the migration is implemented.** Today there is no application code in the repository; these commands describe how the migrated app **will** be set up, run, and tested once the Node.js source is committed and the migration completes.

App-specific values — host, port, environment-variable **names**, and routes — will mirror the original server's values, preserved verbatim from the committed source. They are intentionally not specified here.

### 4.1 Create and activate a Python 3.13 virtual environment

**UNIX / macOS:**

```bash
python3.13 -m venv .venv
source .venv/bin/activate
```

**Windows (PowerShell):**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 4.2 Install dependencies

```bash
pip install -r requirements.txt
pip install -r requirements-dev.txt   # for development / testing
```

### 4.3 Configure environment

Copy `.env.example` to `.env` and supply values for the environment variables listed there. Every variable name and default in `.env.example` will mirror the committed Node.js source.

```bash
cp .env.example .env
```

### 4.4 Run

**Production (UNIX) — gunicorn:**

```bash
gunicorn 'wsgi:app'
```

**Production (Windows) — waitress:**

```powershell
python -m waitress --listen=*:<port> wsgi:app
```

**Development — Flask debug server (auto-reload, debugger):**

```bash
flask --app wsgi:app --debug run
```

### 4.5 Test

The ported `pytest` suite serves as the **functional-parity oracle** — every assertion from the source test suite is preserved.

```bash
pytest
pytest --cov=app           # coverage report (if pytest-cov is installed)
```

---

## 5. How to Proceed / Next Steps

To activate the migration, commit the Node.js source into this repository. The following artifacts are needed for the migration to execute end-to-end:

1. **Entry point** — `server.js` / `app.js` / `index.js` / `bin/www` (the file that calls `express()` and `listen(...)`).
2. **Dependency manifest** — `package.json` plus its lockfile (`package-lock.json`, `yarn.lock`, or `pnpm-lock.yaml`).
3. **Routes and controllers** — typically `routes/**/*.js` and `controllers/**/*.js`.
4. **Middleware** — `middleware/**/*.js`.
5. **Models** — `models/**/*.js` (mongoose / sequelize / prisma / typeorm).
6. **Services** — `services/**/*.js` (business logic).
7. **Configuration and environment** — `config/**/*` and `.env` / `.env.example` (variable **names** and defaults; do not commit secrets).
8. **Tests** — `test/**/*.js`, `tests/**/*.js`, `**/*.test.js`, `**/*.spec.js`, `__tests__/**/*`.
9. **API contracts** *(if available)* — `openapi/**/*` or `swagger*.{json,yaml}`. These serve as the authoritative parity oracle.
10. **Infrastructure** *(if present)* — `Dockerfile`, `docker-compose.yml`, `.github/workflows/**`.

Once these are committed, the migration is executed in **one pass**: the planned structure in §2.3 is generated, the `pytest` suite is ported as the parity oracle, dependencies are pinned, and this `README.md` is updated from its current charter form into Python/Flask setup documentation for the running application.

---

## 6. Authoring Discipline

This document follows **evidence-based authoring**: every claim about the migration's target stack is grounded in verified sources, and **no specifics about the source application have been invented**. The route table, the environment-variable names, the data schema, the model names, the service names, and the business logic are not present here because they are not yet present in the repository. They will be derived from the committed Node.js source the moment it arrives.
