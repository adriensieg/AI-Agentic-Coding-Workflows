# AGENTS.md — FastAPI Web Application

Instructions for AI coding agents working in this repository. Follow these conventions for all generated code.

---

## 1. Bootstrap Philosophy

- Start minimal, evolve incrementally.
- We will start with a backend in FastAPI - and a simple UI in pure javascript, html and css. 

```python
# app.py
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

app = FastAPI()
app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

@app.get("/", response_class=HTMLResponse)
async def index(request: Request):
    return templates.TemplateResponse("index.html", {"request": request})
```

---

## 2. Target Project Structure


```
project/
├── .env                    # Runtime config (git-ignored)
├── requirements.txt
├── main.py             # App factory: wires middleware + routers only
├── core/
│   ├── config.py       # pydantic-settings → typed Settings object
│   └── security.py     # Auth primitives: tokens, sessions, TOTP/QR
├── models/
│   └── <resource>.py   # Pydantic schemas: Read / Create / Update per resource
├── db/
│   └── database.py     # Storage layer (in-memory dict first; swappable for SQLAlchemy)
├── services/
│   └── <resource>_service.py  # Business logic — no HTTP concerns
├──api/
│   └── <resource>.py   # APIRouter — thin handlers that delegate to services
├── README.md
│
├── AGENTS.md                   # Layer 1: how I write code           ← AI knowledge
├── docs/
│   ├── architecture.md         # Layer 1: how the system is designed ← AI knowledge
│   └── features/
│       ├── task-priorities.md  # Layer 2: feature spec (one per feature)
│       └── task-search.md      # Layer 2: feature spec
│
├── opencode.json               # OpenCode config: model, permissions, instructions
└── .gitignore                  # excludes junk from Git AND from agent context
```

**Layering rules (strict):**
- `api/` handles HTTP only — validation, status codes, cookies. No business logic.
- `services/` contain business logic — no `Request`/`Response` objects.
- `db/` handles persistence only. Design so the in-memory store can be swapped for SQLAlchemy without touching services.
- **In-memory state caveat:** while using dict-based storage, run Uvicorn with a single worker (`workers=1`, the default). Multiple workers fork separate processes with divergent copies of the dict, and `--reload` wipes state — both produce erratic behavior. Only scale workers after moving to a real database.
- Dependencies flow one direction: `api → services → db`. Never the reverse.

---

## 3. Configuration

- All settings (API keys, model names, IDs, ports, system prompts) live in `core/config.py` — never hardcoded, never scattered.
- **This project uses Pydantic v2.** Never use v1 syntax (`class Config:`, `@validator`); use `model_config = SettingsConfigDict(...)` and `@field_validator`.
- Use `pydantic-settings` to load `.env` into a typed `Settings` object:

```python
# core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    port: int = 8080
    frontend_url: str = "http://localhost:5173"  # use list[str] if multiple origins are ever needed
    api_key: str = ""

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

settings = Settings()
```

- `.env` is git-ignored. Maintain `.env.example` with placeholder values.
- Agents must never write secrets into source files or commit them.

---

## 4. Logging & Observability

- Configure logging once at startup; use `logger`, not `print`, in application code:

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
)
logger = logging.getLogger(__name__)
```

- Log every request handler entry, external call, and caught exception with enough context to debug without a debugger.
- Log exceptions with `logger.exception(...)` to capture tracebacks.

---

## 5. Running the Server

```python
import uvicorn
from core.config import settings

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=settings.port)
```

- Port is read from config (`PORT` env var, default `8080`).
- Dev command: `uvicorn app.main:app --reload --port 8080`.

---

## 6. Concurrency Rules

**Never block the event loop.**

- **I/O-bound** (network, database, file): use `async def` + `await` with async libraries (`httpx`, `asyncpg`, `aiofiles`).
- **CPU-bound** (heavy math, data processing): offload to a global `ProcessPoolExecutor`:

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

process_pool = ProcessPoolExecutor()

async def handler():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(process_pool, cpu_heavy_fn, arg)
    return result
```

- Blocking sync library with no async alternative → declare the route `def` (not `async def`) so FastAPI runs it in its threadpool.
- Never call `time.sleep()`, `requests.get()`, or blocking DB drivers inside `async def`.

**Background tasks fail silently — always handle their exceptions.**

An unhandled exception inside `asyncio.create_task()` is swallowed until garbage collection: the app keeps running, the task is dead, nothing is logged. Every background task must either wrap its body in `try/except` with logging, or attach a done-callback:

```python
async def _process(item_id: int) -> None:
    try:
        await do_work(item_id)
    except Exception:
        logger.exception("Background task failed for item %s", item_id)

task = asyncio.create_task(_process(item_id))

# Alternative: catch anything that slips through
def _log_task_error(task: asyncio.Task) -> None:
    if not task.cancelled() and task.exception():
        logger.error("Unhandled task exception", exc_info=task.exception())

task.add_done_callback(_log_task_error)
```

- Keep a reference to created tasks (e.g., in a set) — fire-and-forget tasks can be garbage-collected mid-execution.
- For simple post-response work, prefer FastAPI's `BackgroundTasks` dependency over raw `create_task`.

---

## 7. Pydantic Schemas

- Every request/response body is a Pydantic model. Never accept raw dicts.
- Three schemas per resource: `Read` (full), `Create` (required inputs), `Update` (all `Optional`).

```python
from pydantic import BaseModel
from typing import Optional

class DepartureCreate(BaseModel):
    thumbnail: str
    description: str
    platform: str
    departure: str

class DepartureUpdate(BaseModel):
    thumbnail: Optional[str] = None
    description: Optional[str] = None
    platform: Optional[str] = None
    departure: Optional[str] = None
```

- Declare `response_model=` on every route so FastAPI validates and documents output.

---

## 8. CORS

Configured once in `main.py`, origin driven by config — never `allow_origins=["*"]` with credentials:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[settings.frontend_url],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## 9. Type Hints

- Mandatory on every function: parameters and return type.

```python
def get_departure(departure_id: int) -> Optional[Departure]: ...
```

- Type hints power FastAPI's validation, editor autocomplete, and agent comprehension — treat missing annotations as a bug.

---

## 10. Frontend Rules

**The frontend is not a priority in this phase. It exists only to exercise and test the backend.**

- Keep it as minimal as possible: pure HTML, pure CSS, vanilla JavaScript. No frameworks (React/Vue/Svelte), no bundlers, no build step, no npm.
- Use **Bootstrap 5 via CDN** for styling — elegant and minimal, default components, no custom theme work:

```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
```

- Files: `templates/index.html`, `static/script.js`, `static/styles.css` — nothing more unless a feature strictly requires it.
- `script.js` uses plain `fetch()` against the API endpoints; no client-side routing, no state libraries.
- Simple but working: every backend endpoint should be triggerable from the page (forms, buttons, a results table). Function over polish.
- Do not spend effort on responsiveness, animations, accessibility beyond Bootstrap defaults, or visual refinement — backend correctness comes first. The frontend will be rebuilt properly later.

---

## 11. Agent Workflow Rules

- Read the relevant feature spec in `docs/features/*.md` before coding.
- Propose an implementation plan and wait for approval before modifying files.
- Work on a feature branch (`feature/<name>`), never directly on `main`.
- After changes: run the app, verify endpoints via `/docs`, and confirm no regressions before declaring done.
- Keep diffs minimal — do not reformat or refactor unrelated code.
- Update `docs/architecture.md` when structure or design decisions change.

---

## 12. Dependency Management

Rules for agents:

- **Before appending to `requirements.txt`, check the package isn't already listed.** Never create duplicate or conflicting entries; edit the existing line to change a version.
- **Form data / file uploads require `python-multipart`.** Any route using `Form(...)`, `File(...)`, or `UploadFile` will fail (500/422) without it — verify it's in `requirements.txt` before writing such routes.
- Auth features (JWT, TOTP, password hashing) typically require `cryptography`, `pyjwt` or `passlib` — add them explicitly when implementing auth, don't assume they're transitively installed.
- After editing `requirements.txt`, run `pip install -r requirements.txt` and confirm the app starts.

---

## 13. Best-Practice Checklist

- [ ] Use the `lifespan` context manager (not deprecated `@app.on_event`) for startup/shutdown resources:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: open pools, load resources
    yield
    # shutdown: close pools, flush state

app = FastAPI(lifespan=lifespan)
```

- [ ] Use `Depends()` for shared logic (auth, DB sessions) instead of duplicating in handlers.
- [ ] Raise `HTTPException` with correct status codes (`404`, `409`, `422`) — never return error dicts with `200`.
- [ ] Pin dependency versions in `requirements.txt`.
- [ ] Add `pytest` + `httpx.AsyncClient` tests for each new endpoint.
- [ ] Auto-generated docs at `/docs` must remain accurate — schemas and `response_model` keep them so.

---

## 14. AI Agent Pitfalls — Quick Reference

- Never use Pydantic v1 syntax: no `class Config:`, no `@validator`. Use `model_config = SettingsConfigDict(...)` and `@field_validator`.
- Never accept form data or file uploads without confirming `python-multipart` is in `requirements.txt`.
- Never launch `asyncio.create_task()` without exception handling (`try/except` + logging, or `add_done_callback`) — failures are otherwise silent.
- Never fall back to `@app.on_event("startup")` — use the `lifespan` skeleton above.
- Never scale Uvicorn workers or rely on `--reload` persistence while state lives in an in-memory dict.
- Never append duplicate lines to `requirements.txt`.
