# Module 1 — Basic: Claude Code Fundamentals (Detailed Walkthrough)

**Goal:** Get fluent with the core loop — talk to Claude Code, let it write/edit real files, review its work like a teammate's PR.

**Project:** FitTrack API (FastAPI)

---

## Setup

**Environment used:** Claude Code accessed via the "Code" tab in the Claude app (no separate npm install needed for this setup).

**Steps:**
1. Created a project folder (`fittrack-api`) and opened it as a Claude Code session.
2. Confirmed the session was scoped to that folder before doing any work.

**Lesson:** Claude Code operates *inside* a specific folder/project context — always confirm you're pointed at the right one before prompting.

---

## Task 1 — Scaffold the FastAPI Project

**Prompt used (paraphrased):**
> "Set up a basic FastAPI project in this folder. Clean structure with separate folders for models and routes, a `main.py` entry point, and a health-check endpoint at `/health` returning `{"status": "ok"}`. Use a virtual environment and requirements.txt with FastAPI and uvicorn."

**Result — initial file structure:**
```
fittrack_api/
├── main.py
├── models/
│   └── __init__.py
├── routes/
│   ├── __init__.py
│   └── health.py
├── requirements.txt
├── README.md
├── .gitignore
└── venv/
```

**Review checklist applied:**
- ✅ `main.py` actually imported and registered the health router (`app.include_router(health.router)`) — confirmed it wasn't an orphan file
- ✅ `routes/health.py` used correct FastAPI syntax (`APIRouter`, `@router.get("/health")`) and returned exactly `{"status": "ok"}`
- 🔎 Noted: Claude Code added a bonus `/` root endpoint that wasn't explicitly requested — small scope creep, harmless, but a pattern worth always noticing

**Live verification:**
- Ran the server (`python main.py` via the `if __name__ == "__main__"` block)
- Confirmed `http://127.0.0.1:8000/health` returned `{"status":"ok"}`
- Confirmed `http://127.0.0.1:8000/docs` rendered the interactive API docs

**Key lesson:** *Never trust code because it "looks right" — run it and prove it.* This "prompt → generated code → human review → verified execution" loop is the core rhythm repeated at every scale in Claude Code work.

---

## Task 2 — Create CLAUDE.md (Project Memory)

**Why it matters:** `CLAUDE.md` is read automatically by Claude Code at the start of every session in that folder — it's onboarding notes for a new teammate, and it shapes every future interaction, not just the current one.

**Prompt used (paraphrased):**
> "Create a CLAUDE.md file documenting this project's conventions: FastAPI structure with routes/ and models/ folders, use APIRouter with tags for every route file, keep endpoint handlers thin (business logic goes in separate service files), and always include a docstring on each endpoint."

**Result:** A `CLAUDE.md` covering:
- Project structure (including a new `services/` folder, added proactively)
- Route conventions (APIRouter + tags, register in `main.py`)
- Thin-handler pattern (handlers call services, no business logic inline)
- Docstring requirement on every endpoint
- Model conventions (validation lives in Pydantic models, not handlers)
- A worked example route showing the full pattern
- Running/dependency instructions (updated later after the uv migration)

**Lesson:** CLAUDE.md doesn't just document — it **pre-commits** the AI to a style it will hold itself to in later sessions. Also worth verifying OS-specific assumptions it makes (e.g., PowerShell vs. bash commands) rather than assuming they're correct.

---

## Side Quest — Migrate from pip/venv to `uv`

**Rationale discussed:** `uv` (by Astral, the `ruff` team) is now the preferred tool for most new Python projects — much faster than pip, has built-in venv management, Python version management, and a lockfile (`uv.lock`) for reproducible installs. Chosen over pip specifically to fix tooling early, while the project was still small.

**Prompt used (paraphrased):**
> "Migrate this project from pip/venv to uv. Remove the venv folder, set up with uv, convert requirements.txt to a pyproject.toml-based setup, generate a uv.lock file, and update CLAUDE.md's 'Running the app' and 'Dependencies' sections accordingly."

**Issue encountered:** The server process (running with `--reload`) wouldn't stop cleanly through Claude Code's terminal integration; a later `uv --version` check also hung in the same terminal.

**Resolution steps (real troubleshooting, not scripted):**
1. Identified the actual PID holding port 8000 manually: `netstat -ano | findstr :8000`
2. Killed it directly, outside Claude Code: `taskkill /PID <pid> /F`
3. When a later command hung again, verified `uv` was actually installed by running `uv --version` directly in a separate terminal (confirmed working)
4. Told Claude Code directly what was already verified and instructed it to skip the redundant check and proceed

**Key lesson:** When Claude Code (or its terminal integration) gets stuck, don't wait indefinitely — verify manually in a separate terminal and take over. You are the source of truth when its own checks fail or hang.

**Result — new structure:**
```
fittrack_api/
├── main.py
├── models/
├── routes/
│   └── health.py
├── pyproject.toml
├── uv.lock
└── README.md
```

**Verification checklist applied:**
- ✅ Old `venv/` removed
- ✅ `.gitignore` and structure consistent with `uv`'s `.venv/` convention
- ✅ `CLAUDE.md` actually updated — "Running the app" now documents `uv run uvicorn main:app --reload` (with an explanation of why `uv run` auto-syncs from the lockfile), and "Dependencies" documents `uv add` / `uv remove` / `uv sync`
- ✅ Live re-test: server ran, `/health` and `/docs` both responded correctly after migration

**Concept covered — pip vs. uv:**
- pip: created 2008, bundled with Python since 3.4, paired historically with `venv` + separate tools (`pip-tools`, `poetry`, etc.) for locking — ecosystem fragmentation was a long-standing pain point
- uv: released 2024 by Astral, written in Rust, combines venv + pip + lockfile + Python version management into one fast tool

---

## Task 3 — Build the `Workout` Model + Full CRUD

**Spec agreed before prompting** (fields): `id`, `name`, `type`, `duration_minutes`, `calories_burned`, `date`, `notes` (optional)

**Prompt used (paraphrased):**
> "Following the conventions in CLAUDE.md, create a full CRUD API for a Workout resource with these fields... Include a Pydantic model in models/workout.py, a service layer in services/workouts.py with in-memory storage, routes in routes/workouts.py for create/list/get/update/delete, register the router in main.py, thin-handler pattern, docstrings."

### `models/workout.py` — Review Notes
- `WorkoutBase` shared-field pattern, with `WorkoutIn` / `WorkoutUpdate` / `WorkoutOut` built on top — avoids duplicating fields, correctly separates input schema from output schema (output includes `id`, input doesn't)
- Real validation baked in: `min_length=1` on strings, `gt=0` on duration, `ge=0` on calories
- `Field(..., description=...)` on every field — powers the auto-generated `/docs` UI
- 🔎 Flagged: `WorkoutUpdate` inherits all fields as **required**, meaning updates are PUT-style (full replacement) only, not PATCH-style (partial). A silent design decision worth noticing, not necessarily wrong.

### `services/workouts.py` — Review Notes
- Used a **dict keyed by id** (`dict[int, WorkoutOut]`) instead of a list — a better instinct than the list-based approach initially expected, since dict lookups are O(1) vs. a list's O(n) linear scan
- Custom `WorkoutNotFoundError` exception — service layer stays HTTP-agnostic (doesn't raise `HTTPException` itself); that translation is left to the route layer — correct separation of concerns
- Docstring on every function; module-level docstring honestly notes the in-memory storage tradeoff (resets on restart)

### `routes/workouts.py` — Review Notes
- Full checklist passed:
  - ✅ Logic lives in the service layer, not in routes
  - ✅ Handlers are thin (1–3 lines each)
  - ✅ Docstring on every endpoint
  - ✅ `APIRouter(tags=["workouts"])` used
  - ✅ `WorkoutNotFoundError` correctly caught and converted to `HTTPException(404, ...)` in every relevant route — the seam that was specifically being watched for
- Correct REST status codes used: `201 Created` on create, `204 No Content` on delete
- 🔎 Noted for later refactor (Module 2): the try/except-404 pattern is repeated 3 times — a candidate for future de-duplication

### `main.py` — Review Notes
- ✅ Confirmed `workouts.router` properly imported and registered via `app.include_router(workouts.router)`
- 🔎 Noted: it proactively added a docstring to the earlier `root()` endpoint too, without being asked — a sign CLAUDE.md conventions were spreading consistently

### Live End-to-End Test (via `/docs`)
Full CRUD cycle tested manually:
1. POST a new workout → success
2. GET list → new workout appears
3. GET by id → correct single record returned
4. PUT to update → change persisted correctly
5. DELETE → then GET same id → clean `404`, no crash

All steps passed — confirming the full stack (model validation → service logic → route wiring → error handling) actually worked at runtime, not just in review.

---
  
## Module 1 — Skills Demonstrated

- Conversational scaffolding of a real project
- Reading and critiquing AI-generated code like a PR before trusting it
- Live verification of behavior (never trusting code because it "looks right")
- Establishing and testing project memory (`CLAUDE.md`)
- Handling a real tooling migration (pip → uv) including recovering from a stuck terminal
- Understanding and defending architectural choices (layered pattern, MVC comparison, security implications)
- Understanding the project's dependency tree, not just accepting a working install

**Status: ✅ Module 1 Complete**
