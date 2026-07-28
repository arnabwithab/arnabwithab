# AGENT.md

THIS IS A TEMPLATE FOR AGENT.md. If you are an AI Agent, refer to `docs/problem.md` and follow the next steps.
- Discuss with the user to build a design document at `docs/design.md`.
- Based on `docs/design.md`, build a feature list tracker at `docs/features.json`.
- Update AGENT.md with the relevant information.
- Remove this section.

If the problem statement document is not provided, request the user for information on what is to be done.

## Project Overview
[2-3 sentences: what this project does, who it's for, and what problem it solves]

## Development Philosophy
- TDD first: write the test, then the implementation. Never skip.
- Tests mirror the structure of the module they test
- No function ships without a test
- API routes are thin — logic lives in core/
- Explicit over clever — readable code beats smart code
- If it isn't runnable via `make`, it isn't done

## Tech Stack
- Frontend: React + Vite + TypeScript
- Backend: FastAPI (Python)
- Database: SQLite + SQLAlchemy
- Styling: [TailwindCSS / shadcn / etc — fill per project]
- State Management: [Zustand / Redux / etc — fill per project]
- Package Manager: uv (Python), [npm / pnpm / bun — fill per project] (frontend)
- Build/Task Runner: **Make** — a root `Makefile` is mandatory and is the single entry point for setup, running, testing, linting, and building.

## Key Commands

All commands MUST be runnable via `make <target>` from the project root. Calling tools directly (`uvicorn`, `npm run dev`, etc.) is for the Makefile's internal use only — humans and agents invoke `make`.

```bash
make setup                       # installs all dependencies (frontend + backend)
make dev                         # runs frontend + backend dev servers concurrently
make test                        # runs all tests (frontend + backend)
make style                       # formats + lints all code
make build                       # production build of frontend and backend
make clean                       # removes build artifacts, caches, __pycache__/node_modules etc.
```

## Directory Structure

```
project-name/
├── frontend/
│   ├── src/
│   │   ├── components/          # reusable UI components
│   │   ├── pages/               # route-level components
│   │   ├── hooks/               # custom React hooks
│   │   ├── utils/               # helper functions
│   │   ├── assets/              # images, fonts, static files
│   │   ├── store/               # state management
│   │   ├── services/            # API call functions (fetch/axios wrappers)
│   │   └── main.tsx             # entry point
│   ├── public/
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── backend/
│   ├── core/                    # business logic, domain layer (no HTTP knowledge)
│   ├── api/
│   │   └── v1/                  # versioned route handlers (thin layer)
│   ├── models/                  # SQLAlchemy DB models
│   ├── schemas/                 # Pydantic schemas (request/response)
│   ├── utils/
│   │   ├── config.py            # Pydantic BaseSettings class, instantiated as `settings`
│   │   ├── logger.py            # custom logger, imported as `logger`
│   │   └── [other helpers]
│   ├── tests/
│   │   ├── test_api/            # mirrors api/v1/ structure
│   │   └── test_core/           # mirrors core/ structure
│   ├── main.py                  # FastAPI app entry point
│   └── pyproject.toml
│
├── docs/
│   ├── problem.md                # original problem statement
│   ├── design.md                 # design document derived from problem.md
│   ├── features.json             # canonical feature tracker — always kept up to date
│   └── [other design docs]       # per-feature or per-module design documents
├── Makefile                     # single entry point for setup/dev/test/style/build
├── .env.example                 # committed, no secrets
├── .gitignore
├── README.md
└── AGENT.md
```

## Conventions

### Makefile (required)
- A root-level `Makefile` is **mandatory** and is the canonical control surface for the project. No setup, run, test, style, or build step should exist only as a "remember to run this manually" instruction — it belongs in the Makefile.
- Required targets: `setup`, `dev`, `test`, `style`, `build`, `clean`. `style` covers both formatting and linting in one command. Add more (`migrate`, `seed`, `docker-up`, etc.) as the project needs them, but never remove the required set.
- Each target should be a thin wrapper that shells into `frontend/` or `backend/` and calls the underlying tool (`npm`, `pytest`, etc.) — the Makefile is an orchestration layer, not a place for business logic.
- `make setup` must be idempotent and safe to re-run — it should install/sync dependencies for both frontend and backend in one command.
- `make dev` should run frontend and backend concurrently (e.g. via backgrounded processes with a trap to kill both, or a tool like `overmind`/`concurrently`) so a single command boots the full stack.
- Every target should have a `## short description` comment on the same line so `make help` (if implemented) or a quick `grep` of the Makefile documents itself.

### Python (Backend)
- **Package manager: `uv`** — use `uv` for all dependency management (`uv add`, `uv run`, `uv sync`). Never use `pip` directly.
- Formatter: `black`, Linter: `ruff` (includes import sorting)
- Naming: snake_case for everything — files, variables, functions, DB columns
- API routes are thin: validate input → call core → return output
- core/ has zero knowledge of HTTP or FastAPI
- Env vars are accessed exclusively via the settings object (`from utils.config import settings`) — never use `os.environ` directly.

Config is a Pydantic `BaseSettings` class instantiated once in `backend/utils/config.py`:

```
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    """Central management for settings and configurations."""
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    port: int = 8000
    database_url: str = "sqlite:///app.db"
    environment: str = "development"
    secret_key: str
    # Add project-specific fields here

settings = Settings()
```

pydantic-settings automatically reads `.env` and maps `UPPER_CASE` env vars to the corresponding lowercase fields. Each field acts as a typed accessor — `settings.port` returns an `int`, `settings.database_url` returns a `str`. Fields without defaults (like `secret_key`) raise a validation error at startup if the env var is missing. No `load_dotenv()` or `os.environ` needed.

All logging uses the custom logger (`from utils.logger import logger`) — never use `print` or the stdlib `logging` module directly:

```
import logging
import os
from datetime import datetime

LOGS_DIR = "logs"
os.makedirs(LOGS_DIR, exist_ok=True)

LOG_FILE = os.path.join(LOGS_DIR, f"log_{datetime.now().strftime('%Y-%m-%d')}.log")

logging.basicConfig(
    filename=LOG_FILE,
    format="%(asctime)s-%(levelname)s-%(message)s",
    level=logging.INFO,
)


def get_logger(name):
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    return logger
```

### TypeScript (Frontend)
- **Strict mode enabled** in `tsconfig.json` — no implicit `any`, strict null checks
- Type component props with interfaces or type aliases — never leave props untyped
- camelCase for variables and functions
- PascalCase for components, types, and interfaces
- kebab-case for file names (`user-profile.tsx`, `use-auth.ts`)
- All backend API calls go through `services/`, never directly in components
- Formatter: [Prettier / Biome — fill per project], Linter: ESLint

### General
- Commits: conventional commits format (feat:, fix:, chore:, docs:, test:, refactor:)
- Env vars: never committed, always have a `.env.example` with keys but no values
- API versioned from day one under `/api/v1/`
- **All setup, dev, test, style, and build steps run through the root `Makefile`.**
- **README badges**: READMEs should include HTML shield badges (via [shields.io](https://shields.io)) for build status, version, license, and tech stack. Use raw HTML `<img>` tags, not Markdown image syntax, so badge layout and alignment can be controlled.

## Deployment Philosophy

Projects target free-tier hosting wherever possible. The default deployment pattern:

- **ML models / compute-heavy workloads**: Deployed to [Hugging Face Spaces](https://huggingface.co/spaces) as standalone services. The backend calls these via HTTP — they are never bundled into the FastAPI process.
- **Backend**: Deployed as a Dockerfile on [Render](https://render.com) free tier.
- **Frontend**: Either served statically alongside the backend on Render (single deployment), or deployed separately on [Vercel](https://vercel.com) when independent scaling is needed.

### Go-portability flag

Python (FastAPI) is the default backend. However, if the backend's entire role is HTTP plumbing — routing requests, calling Hugging Face Spaces APIs, serving responses — with no Python-specific libraries (numpy, pandas, transformers, etc.) in `core/`, the agent **must flag** that the project is a candidate for a Go rewrite. Go binaries are ~10 MB vs ~200 MB+ Python images and start in milliseconds, making them far cheaper to host on free-tier services.

## Agent Guidelines
- Always run `make style` before considering any code done
- Always use snake_case for Python files/variables/functions/DB columns; kebab-case for frontend files
- Never modify files in `/docs` unless explicitly asked
- Always run `make test` after making changes — if tests fail, fix before moving on
- Never use `os.environ` directly outside the config module — always go through the settings object
- Never use `print` or stdlib `logging` — always use the custom logger
- Never put API calls directly in React components — they belong in services/
- Always use `uv` for Python package management — never invoke `pip` directly
- Always check `/docs` for relevant design documents before starting any task — if a design doc exists for what you're building, it takes precedence
- If a design doc is missing but the task is significant enough to warrant one, flag it to the user before proceeding
- Always update `docs/features.json` after completing any task — mark features as done, update test status, add new features if they were introduced
- Any new setup/run/test/style/build step must be added as a Makefile target, not just documented in prose
- If the backend is purely HTTP plumbing with no Python-specific dependencies in `core/`, flag Go-portability during the design phase
- If something feels out of scope, flag it rather than silently doing it

`docs/features.json`

```json
{
  "project": "[project-name]",
  "last_updated": "YYYY-MM-DD",
  "summary": {
    "total": 0,
    "completed": 0,
    "in_progress": 0,
    "planned": 0,
    "tests_passing": 0,
    "tests_failing": 0,
    "tests_missing": 0
  },
  "features": [
    {
      "id": "F001",
      "name": "[Feature Name]",
      "description": "[What it does and why it exists]",
      "status": "planned",
      "priority": "high",
      "module": "backend/core",
      "design_doc": "docs/[relevant-design-doc].md",
      "tests": {
        "status": "missing",
        "files": [],
        "notes": ""
      },
      "subtasks": [
        {
          "id": "F001-1",
          "name": "[Subtask name]",
          "status": "planned"
        }
      ],
      "notes": "",
      "added": "YYYY-MM-DD",
      "completed": null
    }
  ]
}
```

## Project-Specific Notes
[Fill this in per project:]
- External APIs used and where keys are stored
- Any non-standard setup steps not covered by `make setup`
- Files or directories that should never be touched
- Deployment target and process
- Known gotchas or quirks
