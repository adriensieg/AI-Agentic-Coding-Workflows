# AGENTS.md — FastAPI Web Application

Instructions for AI coding agents working in this repository. Follow these conventions for all generated code.

---

## 1. Bootstrap Philosophy

- Start minimal, evolve incrementally. Phase 1 is a single `app.py`; refactor into the `app/` package only when features demand it.
- Never introduce abstractions (services, repositories, factories) before at least two call sites need them.

### Phase 1 — Minimal skeleton

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

Initial files: `app.py`, `templates/index.html`, `static/script.js`, `static/styles.css`, `requirements.txt`.

---

## 2. Configuration

- All settings (API keys, model names, IDs, ports, system prompts) live in `core/config.py` — never hardcoded, never scattered.
- Use `pydantic-settings` to load `.env` into a typed `Settings` object:

```python
# core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    port: int = 8080
    frontend_url: str = "http://localhost:5173"
    api_key: str = ""

    class Config:
        env_file = ".env"

settings = Settings()
```

- `.env` is git-ignored. Maintain `.env.example` with placeholder values.
- Agents must never write secrets into source files or commit them.

---

## 3. Logging & Observability

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

## 4. Running the Server

```python
# app.py (Phase 1) or app/main.py (Phase 2)
import uvicorn
from core.config import settings

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=settings.port)
```

- Port is read from config (`PORT` env var, default `8080`).
- Dev command: `uvicorn app.main:app --reload --port 8080`.

---

## 5. Target Project Structure (Phase 2)

Refactor into this layout once the app has auth, multiple resources, or >~300 lines:

```
project/
├── .env                    # Runtime config (git-ignored)
├── .env.example            # Template with placeholder values
├── requirements.txt
├── README.md
└── app/
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
    └── api/
        └── <resource>.py   # APIRouter — thin handlers that delegate to services
```

**Layering rules (strict):**
- `api/` handles HTTP only — validation, status codes, cookies. No business logic.
- `services/` contain business logic — no `Request`/`Response` objects.
- `db/` handles persistence only. Design so the in-memory store can be swapped for SQLAlchemy without touching services.
- Dependencies flow one direction: `api → services → db`. Never the reverse.

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

---

## 7. Pydantic Schemas

- Every request/response body is a Pydantic model. Never accept raw dicts.
- Three schemas per resource: `Read` (full), `Create` (required inputs), `Update` (all `Optional`).

```python
from pydantic import BaseModel
from typing import Optional

class DepartureCreate(BaseModel):
    thumbnail: str = "🚆"
    description: str
    platform: str = ""
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

## 10. Agent Workflow Rules

- Read the relevant feature spec in `docs/features/*.md` before coding.
- Propose an implementation plan and wait for approval before modifying files.
- Work on a feature branch (`feature/<name>`), never directly on `main`.
- After changes: run the app, verify endpoints via `/docs`, and confirm no regressions before declaring done.
- Keep diffs minimal — do not reformat or refactor unrelated code.
- Update `docs/architecture.md` when structure or design decisions change.

---

## 11. Best-Practice Checklist

- [ ] Use `lifespan` context manager (not deprecated `@app.on_event`) for startup/shutdown resources.
- [ ] Use `Depends()` for shared logic (auth, DB sessions) instead of duplicating in handlers.
- [ ] Raise `HTTPException` with correct status codes (`404`, `409`, `422`) — never return error dicts with `200`.
- [ ] Pin dependency versions in `requirements.txt`.
- [ ] Add `pytest` + `httpx.AsyncClient` tests for each new endpoint.
- [ ] Auto-generated docs at `/docs` must remain accurate — schemas and `response_model` keep them so.
