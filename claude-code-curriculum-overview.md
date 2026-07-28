# Learning Claude Code — Curriculum Overview

**Format:** Learn-by-doing. Every module is hands-on development work on a single running project, not toy exercises.

**Project:** FitTrack API — a FastAPI backend that grows in complexity across modules.

**Stack:** Python, FastAPI, `uv` (package/environment manager).

**Pace:** Deep dives — each module runs until the concept genuinely sticks, not until it's "technically covered."

---

## Module 1 — Basic: Claude Code Fundamentals

**Goal:** Get fluent with the core loop — talk to Claude Code, let it write/edit real files, review its work like a teammate's PR.

**What was built:**
- FastAPI project scaffold (`main.py`, `routes/`, `models/`)
- `/health` endpoint
- Migration from pip/venv → `uv` (dependency/environment management)
- `CLAUDE.md` — project memory / conventions file
- Full CRUD feature for a `Workout` resource (Model → Route → Service layered architecture)

**Concepts covered:**
- Conversational editing (prompting Claude Code to create/modify files)
- Reviewing AI-generated diffs critically before trusting them ("review before trusting")
- Running commands and verifying behavior live, not just trusting code that "looks right"
- Project memory via `CLAUDE.md` and testing whether Claude Code actually follows its own documented conventions
- Recovering when a command/terminal hangs — verifying manually and taking over rather than waiting indefinitely
- Layered architecture (Model / Route / Service) — what each layer owns, and why
- Comparison to classic MVC architecture
- Security benefits of layered architecture (centralized validation, error handling, auditability)
- Dependency management basics: direct vs. transitive dependencies, `pyproject.toml` vs. `uv.lock`, what Pydantic is and why FastAPI depends on it

**Status:** ✅ Complete

---

## Module 2 — Medium: Working Like a Real Engineer

**Goal:** Use Claude Code on non-trivial, multi-file changes — the way you'd actually work on a growing codebase.

**Planned hands-on tasks:**
1. Add a `User` model + relationships (Workouts belong to Users) — a multi-file change
2. Add JWT auth — practice giving Claude Code a *spec* vs. letting it freelance
3. Use **plan mode** (step back and critique a plan before executing) instead of jumping straight to edits
4. Git workflow: commits, branches, and meaningful commit messages written with Claude Code
5. Debugging exercise: intentionally break something, use Claude Code to diagnose and fix using error logs/tracebacks
6. Refactor a messy function together — practice giving feedback and iterating rather than accepting the first draft

**Concepts to cover:**
- Planning before executing
- Multi-file reasoning
- Git integration
- Debugging workflows
- Iterative feedback loops

**Open decision:** Whether to keep in-memory storage or introduce a real database (e.g., SQLite via SQLAlchemy) as part of this module.

**Status:** ⏳ Not started

---

## Module 3 — Advanced: Scaling the Workflow

**Goal:** Use Claude Code the way experienced users do — automating repeatable work, extending it, and trusting it with bigger, less-supervised chunks of work.

**Planned hands-on tasks:**
1. Write tests with Claude Code (pytest) and use it to fix failing tests it broke
2. Create a **custom slash command** for a repeated task (e.g., "generate a new endpoint + test + docs")
3. Explore **subagents** for a delegated task (e.g., a dedicated "test-writer" persona)
4. Connect an **MCP server** if relevant (optional, depends on interest)
5. Larger unsupervised task: give Claude Code a full feature spec (e.g., a progress-tracking dashboard endpoint with weekly aggregation) and let it work with minimal intervention, then review
6. Performance/cleanup pass: ask Claude Code to profile and suggest improvements

**Concepts to cover:**
- Test-driven workflows
- Custom commands
- Subagents
- MCP (Model Context Protocol)
- Delegation at scale
- Reviewing large autonomous changes

**Status:** ⏳ Not started

---

## Core Habits Being Built Across All Modules

These are the meta-skills the curriculum is really teaching, regardless of module:

- **Review before trusting** — read diffs and generated code like a PR, don't blind-accept
- **Prove it, don't trust it** — always run and test behavior, never assume code "looks right" = "works"
- **Verify manually when stuck** — if Claude Code hangs or a check fails, step outside it and confirm directly rather than waiting indefinitely
- **Understand well enough to explain** — be able to defend/explain what was generated (data structures used, architecture choices, security implications), not just accept it
- **Give specs, not vibes** — the more precise the ask, the easier it is to catch drift
