# AGENTS.md

## Cursor Cloud specific instructions

Durable, non-obvious notes for running/developing this repo in a Cursor Cloud VM.
The startup update script already refreshes dependencies (pytest, `app/backend`
Python deps, and `app/frontend` npm deps). Services are NOT started automatically.

### Product overview / what is in scope

- Primary product: the **dashboard** — a zero-dependency Python stdlib HTTP
  server at `dashboard/server.py` serving `dashboard/dashboard.html`, port 7891.
  This is what the root `docker-compose.yml` and README Quick Start target.
- The React app in `app/frontend` is the same dashboard's source; its
  `.env.development` points `VITE_API_URL` at `http://127.0.0.1:7891`, i.e. the
  React dev server talks to the stdlib dashboard server, NOT the FastAPI backend.
- The FastAPI service in `app/backend` is an OPTIONAL, separate event-driven
  rewrite (needs Postgres + Redis). See the caveat below before relying on it.

### Run the dashboard (primary)

- Standard command: `python3 dashboard/server.py` (see README). It also accepts
  `--host` / `--port`. OpenClaw is NOT required to run it; `install.sh` requires
  OpenClaw but is not needed for local dev.
- Runtime data lives in `data/` and is gitignored. A fresh checkout has almost
  no data, so the board is empty. To get a populated demo board, seed once:
  `cp -n docker/demo_data/*.json data/`.

### Run the React frontend (dev)

- `cd app/frontend && npm run dev` → Vite on `http://localhost:5173`.
- Gotcha: Vite binds IPv6 loopback, so use `localhost:5173`, not
  `127.0.0.1:5173` (the latter refuses the connection). `npm run build` emits to
  `dashboard/dist` (per `vite.config.ts`), which the dashboard server serves.

### Tests / lint

- Run the suite from repo root: `python3 -m pytest tests/` (pytest required).
- Compile check (used by CONTRIBUTING as the lint proxy):
  `python3 -m py_compile dashboard/server.py scripts/kanban_update.py`.
- Known pre-existing failure on `main`: `tests/test_e2e_kanban.py::test_dirty_title_cleaned`
  fails because the `remark` field legitimately keeps `自动预建` while the test
  asserts it is stripped. This is a repo bug, unrelated to environment setup.

### Optional FastAPI backend (`app/backend`)

Only needed if working on the event-driven architecture. It requires Postgres
and Redis. The intended dev path is `app/docker-compose.yml`; when running
natively instead, note:

- Alembic config `app/alembic.ini` hardcodes role `ac` / password `ac_dev_2024`
  and DB `agents_company`. Create that role + DB, then run migrations from the
  `app/` dir: `python3 -m alembic upgrade head`.
- Start the API from `app/backend` with the DB/Redis overrides, e.g.
  `DATABASE_URL_OVERRIDE=postgresql+asyncpg://ac:ac_dev_2024@127.0.0.1:5432/agents_company`
  `REDIS_URL=redis://127.0.0.1:6379/0 uvicorn app.main:app --host 0.0.0.0 --port 8000`.
- `/health`, `/api`, and `/api/agents` work. Known pre-existing bug: `/api/tasks`
  raises `UndefinedColumnError: column tasks.id does not exist` because the
  `001_initial` migration schema (`task_id`, `trace_id`, `assignee_org`, ...) does
  not match the ORM `Task` model (`id`, `org`, `official`, ...). This is a
  code-level model/migration mismatch, not an environment problem.
