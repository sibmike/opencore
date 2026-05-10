# Dev Environment: FastAPI + Next.js Monorepo

Full-stack local setup for a Python/FastAPI backend and Next.js frontend sharing a single monorepo, targeting AWS services.

---

## Quick start

Install dependencies and start both servers before writing any code. Run each in a separate terminal.

```bash
# Backend
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000

# Frontend
cd frontend
npm install
npm run dev
```

The backend listens on `http://localhost:8000`. The Next.js dev server listens on `http://localhost:3000`. Hot-reload works independently on each layer — a Python change restarts uvicorn without touching the Node process, and vice versa.

Copy `.env.example` to `.env` in both `backend/` and `frontend/` before starting. Never commit `.env` files.

---

## Backend setup: Python venv

Use a virtual environment scoped to `backend/`. Never install backend dependencies into a system Python or a conda base environment — cross-contamination causes silent import failures when the same package name resolves to a different version at test time versus run time.

```bash
cd backend
python -m venv .venv
# macOS / Linux
source .venv/bin/activate
# Windows (PowerShell)
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Always invoke pytest through the venv interpreter, not a bare `pytest` command, to guarantee the correct environment is active:

```bash
backend/.venv/bin/python -m pytest tests/ -x -q        # macOS/Linux
backend/.venv/Scripts/python.exe -m pytest tests/ -x -q # Windows
```

Set `COGNITO_VERIFY_TOKENS=false` in `conftest.py` (or your test `.env`). Without this flag, every test that exercises a JWT-protected route fails because there is no live Cognito user pool in the local environment.

Empty `STEP_FUNCTION_ARN` (or equivalent async-trigger env var) is the standard local-dev guard: the service call becomes a no-op rather than failing. Never raise on a missing ARN during local development; let the guard swallow it silently.

---

## Frontend setup: Node version manager

Pin the Node version at the project root (`.nvmrc` or `.node-version`). The Next.js App Router requires Node 20+. Mismatched Node versions between developers and CI cause build failures that are hard to diagnose because the error appears in a compiled output, not a source file.

```bash
# With nvm (macOS/Linux)
nvm install
nvm use
cd frontend
npm install
npm run dev

# With fnm (cross-platform, faster)
fnm use
cd frontend
npm install
npm run dev
```

Run all frontend commands from `frontend/`. `npm run dev`, `npm test`, and `npx vitest` all require `node_modules/` to be present in the working directory.

---

## Docker Compose: local AWS service stand-ins

Use Docker Compose to run DynamoDB Local and PostgreSQL alongside the FastAPI backend. This eliminates the need for live AWS credentials during development and keeps local state isolated from staging.

```bash
# Start DynamoDB Local + PostgreSQL + backend
docker compose up

# Start DynamoDB Local only (run the backend outside Docker)
docker compose up dynamodb-local
```

Set `DYNAMODB_ENDPOINT_URL=http://localhost:8000` in your backend `.env` to redirect DynamoDB calls to DynamoDB Local. Omit this variable (or point it at the real AWS endpoint) to connect to live AWS. The FastAPI client code does not need to change — aioboto3 picks up the endpoint override from the env var.

DynamoDB Local does not enforce IAM. Any credential string is accepted. Use `AWS_ACCESS_KEY_ID=local AWS_SECRET_ACCESS_KEY=local` when running against DynamoDB Local to avoid credential-not-found errors.

---

## Git worktrees: frontend cannot run from a worktree

Git worktrees check out a second branch into a separate directory. The Python backend works from a worktree; the Next.js frontend does not.

**Frontend — do not run npm commands from a worktree.**

`node_modules/` is not copied into a worktree. Running `npm run test`, `npx vitest`, or `npm run dev` from the worktree's `frontend/` directory fails immediately with `MODULE_NOT_FOUND`. Do not attempt to fix this by symlinking or junctioning `node_modules/` — on Windows, junctions do not behave the same as directories for module resolution, and this approach wastes time without solving the underlying problem.

Always run frontend commands from the main repo's `frontend/` directory where `node_modules/` exists. New frontend test files written in a worktree are only testable after the worktree is merged back to the integration branch and you run tests from the main repo.

**Backend — Python tests run from a worktree.**

Python resolves packages through the venv path, not a relative `node_modules/`-style directory. Use the absolute path to the venv interpreter when invoking pytest from inside a worktree:

```bash
# From inside the worktree
/absolute/path/to/backend/.venv/bin/python -m pytest backend/tests/ -x -q      # macOS/Linux
/absolute/path/to/backend/.venv/Scripts/python.exe -m pytest backend/tests/ -x -q # Windows
```

The worktree carries its own `backend/` directory, so new test files written in the worktree are immediately runnable without merging first.

---

## Cross-platform pitfalls (Windows)

**PowerShell activation.** On Windows, the default `ExecutionPolicy` blocks `.ps1` scripts. Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once, then activate with `.venv\Scripts\Activate.ps1`.

**Path separators.** Python's `subprocess` calls and `os.path` constructions using hardcoded `/` forward slashes work on Windows in most cases, but shell invocations via `subprocess.run(shell=True)` break when paths contain spaces and are not quoted. Prefer `pathlib.Path` over string concatenation for all file paths in tests.

**conda + npm PATH conflict.** If conda is installed, its `Scripts/` directory often appears before the project's Node version manager in `PATH`. The result is that `npm` resolves to a conda-managed Node binary at a different version than the one pinned in `.nvmrc`. Verify with `where npm` (Windows) or `which npm` (macOS/Linux) before running any frontend commands. If the wrong binary appears, activate your Node version manager explicitly before running npm.

**Line endings.** Configure `.gitattributes` to force `text eol=lf` for Python and JavaScript source files. Mixed CRLF/LF in the same file causes `pytest` to misreport line numbers and can break shebang-based scripts.

**`node_modules/` junctions.** Do not use `mklink /J` to share `node_modules/` between the main repo and a worktree. See [Git worktrees](#git-worktrees-frontend-cannot-run-from-a-worktree) above.

---

## Cross-references

- `practices/testing-fullstack.md` — pytest + moto async fixtures; Vitest + MSW v2; `COGNITO_VERIFY_TOKENS=false` in conftest; Playwright E2E setup
- `practices/deployment-aws.md` — `NEXT_PUBLIC_*` build-time vs runtime env vars; App Runner + Amplify deployment; env var split between `apprunner.yaml` and `.env.production`
- `practices/debugging-fullstack.md` — App Runner health-check diagnostics; CORS vs missing DynamoDB table misdiagnosis; TanStack Query three-branch guard

---

<!-- provenance: fragments F-041 (dev-environment.md § Git Worktrees), F-042 (dev-environment.md § Frontend CANNOT run from worktree), F-043 (dev-environment.md § Backend CAN run from worktree), F-116 (README.md § Quick Start), F-117 (README.md § Local Development with Docker). Stack module: full-stack-web-aws v0.1.0. Written 2026-05-10. -->
