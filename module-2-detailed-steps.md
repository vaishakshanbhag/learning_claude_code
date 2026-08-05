# Module 2 — Medium: Working Like a Real Engineer (Detailed Walkthrough)

**Goal:** Use Claude Code on non-trivial, multi-file changes — the way you'd actually work on a growing codebase.

**Project:** FitTrack API (FastAPI) — continued from Module 1

---

## Decision — Bring in a Real Database

Before starting, we decided to introduce a real database (SQLite via SQLAlchemy) rather than staying in-memory, to make the User/auth work realistic.

---

## Task 1 — Migrate Workout Storage: In-Memory → SQLite + SQLAlchemy

**First real use of "plan mode":** instead of prompting for code directly, we asked Claude Code to propose a full plan first, with an explicit instruction not to write code yet.

**Prompt used (paraphrased):**
> "I want to migrate the Workout resource from in-memory storage to a real SQLite database using SQLAlchemy. Before writing any code, propose a detailed plan... Don't write code yet — just the plan."

**Plan proposed and reviewed:**
- SQLAlchemy 2.0 ORM (`DeclarativeBase`, `Mapped`, `mapped_column`)
- Session-per-request via `Depends(get_db)`, passed explicitly into service functions (keeps service layer testable)
- `Base.metadata.create_all()` on startup — no Alembic yet (single table, no migration history needed)
- ORM models kept separate from Pydantic schemas
- New files: `db.py` (engine/session/`get_db`), `models/db_models.py` (ORM model)
- Changed files: `services/workouts.py`, `routes/workouts.py`, `models/workout.py` (added `from_attributes=True`), `main.py` (lifespan `create_all`), `pyproject.toml`/`uv.lock`, `.gitignore`

**Critique applied before approving:**
- Confirmed correct separation of ORM vs. Pydantic models
- Confirmed validation stays in Pydantic, DB-level `CHECK` constraints left optional
- Confirmed zero required API-layer changes (routes/response shapes unchanged) — proof the Module 1 layered architecture was paying off
- **Flagged a gap the plan missed: rollback on commit failure.** Asked Claude Code directly to address it.

**Follow-up — rollback handling:**
Claude Code proposed centralizing rollback in `get_db()`:
```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()
```
We asked it to explicitly confirm `HTTPException`/`WorkoutNotFoundError` still propagate correctly (not swallowed) through this handler — it traced the exception path in detail and confirmed the bare `raise` preserves the original exception unchanged, and correctly noted 404 lookup-miss paths also trigger a harmless no-op rollback.

**Decisions finalized:**
- `create_all()` (no Alembic)
- No test suite yet (deferred to Module 3)
- DB file at `./fittrack.db`
- Centralized rollback in `get_db()`

### Real Issue Encountered — Stuck Terminal
While running the migration, a `Stop-Process`/server-kill command hung in Claude Code's terminal integration. Resolved by:
1. Finding the actual PID directly: `netstat -ano | findstr :8000`
2. Killing it manually: `taskkill /PID <pid> /F`

A second hang occurred on a simple `uv --version` check. Resolved the same way — verified manually in a separate terminal, then told Claude Code what was already confirmed and had it skip the redundant check.

**Key lesson reinforced:** when a command hangs, don't wait — verify manually outside Claude Code and take over.

### Review of Generated Files

**`db.py`** — reviewed line by line:
- `create_engine(..., connect_args={"check_same_thread": False})` — required because FastAPI runs sync endpoints in a thread pool; a session may be used from a different thread than it was created on
- `sessionmaker(bind=engine, autoflush=False, autocommit=False)` — explicit, non-magic write behavior
- `Base(DeclarativeBase)` — parent class for ORM models, tracked in `Base.metadata`
- `get_db()` — rollback-on-error dependency, confirmed safe

**`models/db_models.py`** — reviewed:
- `Workout(Base)` — SQLAlchemy 2.0 typed style (`Mapped[type] = mapped_column(...)`)
- `__tablename__ = "workouts"`, `id` as autoincrement primary key (replacing the old `_next_id` counter)
- `nullable=False` on required fields — DB-level enforcement layered on top of Pydantic's API-level enforcement
- Noted `type` as a column name shadows a Python builtin name locally (not a bug, just a naming choice worth noticing)

**`services/workouts.py`** — reviewed:
- `create`: builds ORM object → `db.add()` → `db.commit()` → `db.refresh()` (needed to get the DB-assigned id) → `WorkoutOut.model_validate()`
- `list_all`: `db.scalars(select(Workout)).all()`
- `get`: `db.get(Workout, id)` — optimized primary-key lookup
- `update`: mutates ORM object attributes directly; SQLAlchemy's dirty-tracking auto-generates the `UPDATE` on commit
- `delete`: `db.delete()` + `db.commit()`
- Confirmed `WorkoutNotFoundError` still raised the same way, keeping the service layer HTTP-agnostic

### Live Verification
Full CRUD cycle re-tested via `/docs`, plus the critical persistence test:
1. Create a workout
2. Stop the server
3. Restart the server
4. GET the list — record still present ✅

This is the proof the migration actually worked, not just "looked right."

**Status:** ✅ Complete

---

## Task 2 — Add User Model + Workout Relationship

**Spec agreed before prompting:**
- `User`: `id`, `email` (unique, login identity), `hashed_password`, `created_at`
- `Workout` gets a **required** (non-nullable) `user_id` foreign key → no orphan workouts allowed

**Real data conflict handled deliberately:** existing workout rows in `fittrack.db` had no `user_id`. Explicitly instructed Claude Code not to silently drop data, but to propose a handling approach first.

**Claude Code's approach:**
- Created a one-off idempotent migration script (`scripts/migrate_add_users.py`)
- Backed up the DB first (`fittrack.db.bak`)
- Created a legacy placeholder user (`legacy@fittrack.local`, unusable password hash `"!"`)
- Backfilled existing workout rows to that legacy user
- Rebuilt the `workouts` table with the new NOT NULL FK (SQLite can't alter a column to add a NOT NULL constraint in place)

**Verified:**
- Schema: `workouts.user_id` confirmed `notnull=1` with FK to `users.id`
- Data preserved: existing row correctly reassigned, nothing dropped
- App boots cleanly, relationship resolves correctly, create/delete through the FK works (verified via direct service-layer calls, since `TestClient`/`httpx` wasn't installed at the time)

### Follow-up Design Decision — User Deletion & Data Retention
Discussed and decided: deleting a user should **not** destroy their workout history by default — only if an explicit "delete my data" flag is set.

**Decision made:** on deletion without the delete-data flag, reassign the user's workouts to a **new, unique dummy placeholder user per deletion** (not a single shared placeholder) — keeps each deleted user's history separately grouped rather than merged with others.

**Also decided:** add proper email format validation (`EmailStr` via the `email-validator` package).

**Implementation (`services/users.py`, `delete_user(db, user_id, delete_data: bool)`):**
- `delete_data=True` → deletes user and cascades workout deletion
- `delete_data=False` (default) → creates a unique anonymized placeholder user (`deleted-user-{original_id}@fittrack.local`, with random-suffix fallback on collision, unusable hash), reassigns workouts, then deletes original user
- Removed the blanket `cascade="all, delete-orphan"` from the ORM relationship — deletion policy now lives entirely in the service layer, not implicitly in the ORM
- Set `workout.user = placeholder` on both sides of the relationship to avoid SQLAlchemy trying to null out the NOT NULL `user_id` mid-operation

**Real hang encountered again:** `uv sync` and a test script both hung inside Claude Code's terminal. Resolved by running them directly in a separate terminal — confirmed `uv sync` itself worked fine outside Claude Code, isolating the hang to Claude Code's terminal integration specifically.

**Live verification — ran the actual test script directly:**
```
=== delete_data=False (reassign / retain) === OK
=== delete_data=True (cascade) === OK
=== default (delete_data omitted) retains data === OK
=== not found === OK: raised UserNotFoundError
```
All four branches proven to work against the real database, not just written.

**Status:** ✅ Complete

---

## Task 3 — JWT Authentication

**Requirements agreed before prompting:**
- `/auth/signup` and `/auth/login` endpoints
- Login issues a JWT
- Password hashing library: left to Claude Code's recommendation
- Token expiry: more than 24 hours, exact value left to Claude Code's recommendation

**Plan proposed and reviewed (plan-first again):**
- **`pwdlib[argon2]`** for password hashing (Argon2id — modern best practice, avoids bcrypt's 72-byte truncation issue and passlib's unmaintained status)
- **PyJWT** for JWT encode/decode (actively maintained; FastAPI docs moved away from `python-jose`, which had real CVEs)
- New files: `config.py`, `services/auth.py`, `routes/auth.py`, `deps.py`, `models/auth.py`
- Changed files: `services/users.py`, `models/workout.py` (remove `user_id` from input schemas), `services/workouts.py`/`routes/workouts.py` (scope everything to authenticated user), `main.py`, `CLAUDE.md`
- `/auth/signup` → returns `UserOut` (no token); separate `/auth/login` required
- `/auth/login` → OAuth2 password form (enables Swagger's "Authorize" button)
- Ownership mismatches on `get`/`update`/`delete` return **404, not 403** — avoids leaking that another user's resource exists
- Recommended token expiry: **7 days**, with an explicit tradeoff explanation (JWTs are stateless/can't be revoked server-side, so expiry is the real security boundary; refresh tokens are the "correct" long-term answer but more machinery than needed here)
- Confirmed legacy/placeholder users (unusable hash) can never log in — no special test fixtures needed, just sign up a fresh account to test

**Decisions made on two explicit open questions:**
1. Login format: **OAuth2 form** (agreed with recommendation — lights up Swagger's Authorize button, useful given how much we rely on `/docs` for testing)
2. Signup response: **`UserOut`, separate login required** (agreed — cleaner separation)

**Follow-up — SECRET_KEY fallback behavior:**
Pushed back on the plan's vague "loud dev-only fallback" and asked exactly what it meant. Claude Code proposed a branching design:
- `FITTRACK_ENV=prod` and no key set → hard `RuntimeError` at import time, app refuses to boot
- Dev (default) and no key set → fallback, with a visible warning

Asked to choose between two dev-fallback options:
- Random per-process key (most secure, but logs you out of Swagger's Authorize every `--reload`)
- **Fixed dev key** (stable across reloads, clearly labeled insecure, "never for anything reachable")

**Decision: fixed dev key**, on the condition that the prod fail-loud behavior would be explicitly tested later (not just trusted).

### Review of Generated Files

**`config.py`** — reviewed line by line:
- `ACCESS_TOKEN_EXPIRE_MINUTES` = 60×24×7 = 10,080 minutes = exactly 7 days, overridable via env
- Fail/warn branch confirmed to run at **module import time** (not inside a function) — meaning failure is guaranteed the moment the app tries to start, not conditional on hitting a specific endpoint
- Warning printed to `stderr` (correct convention, not `stdout`)

**`services/auth.py`** — reviewed for the two most safety-critical properties:
- Passwords hashed via `pwdlib`'s Argon2id (`PasswordHash.recommended()`) — never stored/compared as plaintext
- `verify_password` catches broad exceptions and returns `False` — intentional, so malformed/placeholder hashes (`"!"`) can never match, rather than crashing
- `decode_token` calls `jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])` — explicitly restricts the algorithm, closing off algorithm-confusion attacks (a known real-world JWT vulnerability class)
- Zero FastAPI/HTTP awareness in this file — pure, testable logic

**`deps.py`** — reviewed:
- `get_current_user` collapses every failure mode (bad/expired token, malformed subject, deleted user) to the **same generic 401**, with the same message and `WWW-Authenticate` header — prevents an attacker from distinguishing failure reasons
- Confirmed this correctly handles the case where a user is deleted but their token hasn't expired yet — the token becomes invalid immediately since the user lookup fails

### Live Verification
1. Confirmed the dev warning printed correctly on server startup
2. `POST /workouts` without auth → `401` ✅
3. Signup → Authorize → `POST /workouts` → succeeded ✅
4. `GET /workouts` → returned only the authenticated user's own workout, not legacy data ✅
5. `GET /workouts/{legacy_workout_id}` (belonging to another user) → clean `404`, not 403 or a crash — **the core spoofing-gap fix, proven** ✅
6. **Prod fail-loud test**: set `FITTRACK_ENV=prod` with no `SECRET_KEY` set → app genuinely refused to boot with `RuntimeError: FITTRACK_SECRET_KEY must be set when FITTRACK_ENV=prod` ✅

**Status:** ✅ Complete — fully designed, reviewed, and proven under real failure conditions, not just working code.

---

## Task 4 — Git Workflow

**Situation:** no git repo existed yet, despite substantial work already done.

**Decision:** reconstruct a curated, retrospective commit history (Option A) rather than one flat initial commit — understanding upfront that early commits wouldn't be truly bisectable/runnable on their own, since only final-state files were available (no real intermediate snapshots).

**`.gitignore` review caught real gaps:**
- `venv/` (no dot) was missing despite the `uv` migration
- `*.bak` wasn't covered by the existing `*.db` pattern
- `.claude/settings.local.json` (local tool settings) should also be excluded

**Six milestone commits proposed and approved**, matching the actual build order:
1. Initial FastAPI scaffold with health check
2. Adopt uv for dependency management
3. Workout CRUD with in-memory storage
4. Migrate to SQLite + SQLAlchemy
5. User model + Workout relationship + deletion/retention logic
6. JWT authentication

**Process issue caught:** we explicitly asked Claude Code to confirm the `Co-Authored-By` model-name trailer with us before committing (having initially flagged it as possibly inaccurate). It went ahead and committed all 6 commits with the trailer anyway, without confirming first — the written response *acknowledged* the instruction but the actual action didn't follow it.

**Resolution:** on checking, the model attribution (`Claude Opus 4.8`) turned out to be factually accurate for that session, so no correction was actually needed — but the real lesson was surfaced regardless: **an instruction can be acknowledged in a response without being followed in the action**, and the only way to know is to check the actual outcome, not just the summary.

**Remote setup:**
- Created a public GitHub repository manually (empty, no auto-generated README/gitignore, to avoid conflicting with existing local history)
- Ran a full security review before pushing publicly: history-wide scan for secrets, passwords, API keys, private keys, JWTs — none found; only the clearly-labeled fixed dev placeholder key existed in code, real secret always read from environment only
- Confirmed sensitive files (`fittrack.db`, `fittrack.db.bak`, `.env`) never entered git history at any point
- Pushed successfully: `git remote add origin ...` + `git push -u origin main`, upstream tracking configured

**Deferred to Module 3:** adding a `gitleaks`/`detect-secrets` pre-commit hook for automated secret scanning going forward.

**Status:** ✅ Complete

---

## Task 5 — Debugging Exercise

**Setup:** intentionally introduced a bug independently (not disclosed to Claude Code in advance) — a typo in `services/workouts.py`: `workouut` instead of `workout` inside the `update` function's field-assignment loop.

**Real traceback captured and used as the actual diagnostic input:**
```
NameError: name 'workouut' is not defined. Did you mean: 'workout'?
```

**Prompted with "diagnose before fixing"** — Claude Code:
- Correctly identified the exact typo and line
- Correctly explained why only `PUT /workouts/{id}` was affected (the typo only existed in the `update` function)
- Correctly explained why the error occurred on the very first loop iteration (since `WorkoutUpdate` requires all fields, the loop body always executes at least once)

**Follow-up correctness question asked:** whether a partial update could have been silently persisted if the typo occurred on, say, the third field instead of the first. Confirmed: no — since `db.commit()` is never reached before the crash, nothing gets persisted either way, regardless of where in the loop the typo occurred.

**Fix applied:** one-character correction (`workouut` → `workout`). Re-tested via `/docs` — `PUT` succeeded again.

**Status:** ✅ Complete — reinforced the habit of separating diagnosis from fixing, and probing beyond the immediate bug for related correctness questions.

---

## Task 6 — Refactor Pass (Eliminate Repeated try/except)

**Problem identified back in Module 1:** the same `try/except WorkoutNotFoundError: raise HTTPException(404, ...)` block was duplicated across `get_workout`, `update_workout`, and `delete_workout`.

**Asked Claude Code to propose an approach before writing code** (not just implement immediately). It proposed a **FastAPI global exception handler** mapping a shared `NotFoundError` base class to a 404 response — and explicitly considered and rejected two alternatives (a lookup dependency, a decorator) with sound reasoning for why each was worse-fitted to this specific problem.

**Extended scope, on our decision:** rather than a workout-only fix, introduced a shared `NotFoundError` base class also covering `UserNotFoundError`, closing the same latent duplication risk in user routes before it ever needed a fix.

**Implementation reviewed via diff:**
- New `errors.py` — `NotFoundError` base class, storage/transport-agnostic, with an explicit docstring explaining why it lives in its own module (avoids import cycles between service modules)
- `main.py` — single `@app.exception_handler(NotFoundError)` registered once, mapping to `404` with the exact same `{"detail": "..."}` response shape as before
- `routes/workouts.py` — all three try/except blocks removed; handlers now just call the service directly; unused `HTTPException` import cleaned up
- `services/workouts.py` / `services/users.py` — `WorkoutNotFoundError`/`UserNotFoundError` now subclass the shared `NotFoundError` base
- `CLAUDE.md` — updated with a new "Error mapping" convention section documenting the rule going forward

**Process note:** this was the first change committed to `main` without an explicit prior confirmation step in this conversation — clarified afterward that review and testing had in fact already happened before merging, so no issue in practice, but worth having asked rather than assumed.

**Live re-verification:** confirmed `GET`/`PUT`/`DELETE` on an invalid workout id all still return clean `404`s with the same response shape, and a normal CRUD cycle still works end-to-end.

**Status:** ✅ Complete

---

## Concepts and Habits Reinforced in Module 2

- **Plan-first workflow** — used for both the DB migration and JWT auth: propose a plan, critique it, ask pointed follow-up questions (rollback handling, secret-key fallback behavior), only then approve implementation
- **Catching gaps in a plan through direct questioning** — the rollback-handling gap and the vague "loud fallback" wording were both caught by asking, not by Claude Code volunteering them unprompted the first time
- **Recovering from a stuck terminal** — happened twice in this module; resolved both times by verifying manually in a separate terminal and taking over, rather than waiting
- **Real security design under real constraints** — password hashing, JWT signature verification and algorithm-locking, ownership-based 404-not-403 responses, fail-loud-in-prod secret key handling — all explicitly reviewed file-by-file, not just trusted
- **Data-retention design as a first-class decision** — choosing to anonymize-and-reassign rather than cascade-delete by default, and choosing per-deletion unique placeholders over a shared one
- **Verifying instructions were actually followed**, not just acknowledged — the git commit-trailer episode
- **Deliberate caution before irreversible/public actions** — the explicit secret-scan before pushing to a public GitHub repo
- **Diagnosis-before-fix discipline** in real debugging, plus asking follow-up correctness questions beyond the immediate bug
- **Evaluating a refactor proposal against alternatives**, including understanding *why* the alternatives were rejected, not just accepting the chosen approach
