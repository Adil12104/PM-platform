# PM Platform

A full-stack project management app — workspaces, projects, Kanban boards,
subtasks, comments, real-time updates, notifications, and search. Built as
a learning project with **React → FastAPI → PostgreSQL** and nothing else
(no Redis, no Mongo, no third-party auth/storage services).

---

## ⚠️ You're upgrading from Phase 1 — read this first

If you already have a database from the earlier auth-only version of this
project, **your old `alembic_version` table points at a migration file that
no longer exists** in this codebase (the schema grew from 1 table to 13).
Reusing that database will make Alembic fail with "can't locate revision."

Reset before continuing — this project only ever held test data, so this is safe:

```sql
-- via psql or pgAdmin's query tool, connected to your existing database
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
```

Then follow the setup steps below from "Database migration."

---

## Tech stack

| Layer | Choice |
|---|---|
| Frontend | React + TypeScript + Vite, React Router, TanStack Query, Tailwind CSS, dnd-kit, Recharts |
| Backend | Python + FastAPI, SQLAlchemy 2.0, Pydantic v2, python-jose (JWT), Alembic |
| Database | PostgreSQL — the only datastore. File attachments are metadata-in-Postgres + bytes on local disk. |
| Real-time | Native FastAPI WebSockets — no Redis or message broker |

## Architecture

```
React (Vite dev server, :5173)
   |  REST (axios + JWT bearer token) + WebSocket (native browser API)
   v
FastAPI (uvicorn, :8000)
   |  SQLAlchemy ORM
   v
PostgreSQL
```

Two WebSocket channels:
- `/ws/projects/{project_id}` -- a "room" per project. Every task/subtask/
  comment/attachment mutation broadcasts here, so everyone viewing that
  project's board sees changes live.
- `/ws/notifications` -- a personal channel per logged-in user, for the
  navbar's unread badge and toasts regardless of what page they're on.

Since there's exactly one backend process for local dev, both channels are
just in-memory Python dicts (see `backend/app/websocket/manager.py`) -- no
Redis pub/sub needed. (If this were ever run across multiple backend
processes, that in-memory approach would stop working -- see the comment in
that file for why.)

## Folder structure

```
pm-platform/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app, router wiring, startup hook
│   │   ├── core/                # settings (config.py), JWT + hashing (security.py)
│   │   ├── database/            # SQLAlchemy engine/session (database.py)
│   │   ├── models/               # ORM models, one file per entity + enums.py
│   │   ├── schemas/              # Pydantic request/response shapes
│   │   ├── api/                  # FastAPI routers (one per resource) + deps.py (auth/authorization)
│   │   ├── services/              # business logic -- routes stay thin, services hold the rules
│   │   └── websocket/             # connection manager + the two WS routes
│   ├── alembic/                  # migration environment (env.py wired to app settings/models)
│   ├── scripts/seed.py            # demo data generator
│   └── requirements.txt
├── frontend/
│   └── src/
│       ├── api/client.ts          # axios instance: attaches JWT, handles 401
│       ├── types/domain.ts        # all shared TS types
│       ├── services/               # one file per resource, thin wrappers over axios
│       ├── hooks/                  # useAuth, useWorkspace, useWebSocket, useToast
│       ├── components/             # Sidebar, Topbar, KanbanBoard, TaskModal, etc.
│       ├── layouts/DashboardLayout.tsx
│       └── pages/                  # one per route
├── docker-compose.yml              # Postgres only
└── README.md
```

## Database schema

13 tables. Every relationship exists for a specific reason:

- **users** -- identity/auth only.
- **workspaces** / **workspace_members** -- a workspace is the top-level
  container; `workspace_members` is the join table that also carries the
  `role` (OWNER/ADMIN/MEMBER) -- every authorization check in the app
  ultimately queries this table.
- **projects** / **project_members** -- a project belongs to exactly one
  workspace (FK, `ondelete="CASCADE"`). `project_members` is a second,
  narrower membership layer -- a project member must already be a
  workspace member (enforced in `project_service.py`, not the DB, since
  that's a cross-table rule).
- **tasks** -- the Kanban card. Has a `position` integer (unique-ish within
  `project_id + status`) so drag-and-drop order survives a refresh.
- **subtasks**, **labels** / **task_labels** (the many-to-many join
  between tasks and labels), **comments**, **attachments** (metadata only
  -- bytes live on disk under `backend/uploads/`) -- all hang off `tasks`
  with `ondelete="CASCADE"`, so deleting a task cleans up everything
  attached to it.
- **notifications** -- one row per (recipient, event). Uses nullable
  `related_task_id`/`related_project_id` integer pointers rather than a
  separate nullable FK per possible target type, since a notification can
  be about very different kinds of things.
- **activities** -- an append-only audit log, scoped to `project_id` so a
  project's timeline is a single indexed query.

Migrations are managed with Alembic -- see "Database migration" below.
`Base.metadata.create_all()` is never used in this project on purpose: it
can only add new tables, never alter existing ones, which breaks the
moment you need to change a column on data that already exists.

## Authorization rules (enforced server-side, not just hidden in the UI)

| Action | Who can do it |
|---|---|
| Create a workspace | Any authenticated user (becomes OWNER) |
| Rename / edit a workspace | OWNER or ADMIN |
| Delete a workspace | OWNER only |
| Invite a member | OWNER or ADMIN (only OWNER can hand out OWNER/ADMIN roles) |
| Remove a member | OWNER can remove anyone (except the last OWNER); ADMIN can only remove MEMBERs; anyone can remove themselves |
| Change a member's role | OWNER only, can't demote the last remaining OWNER |
| Create a project | Any workspace member |
| Edit / delete a project | Workspace OWNER or ADMIN |
| Create / edit / move / delete tasks, subtasks, labels | Any workspace member -- Kanban boards are collaborative by design |
| Edit / delete a comment | Only the comment's author |

Every one of these is checked in `backend/app/api/deps.py` and
`backend/app/services/*.py` -- a request that skips the frontend entirely
(curl, Postman, browser devtools) and hits the API directly gets the same
403/404 responses. All authorization edge cases above were exercised in a
manual end-to-end test run against the real API before this was handed
off (a MEMBER genuinely cannot delete a workspace or rename a project by
calling the API directly).

## Setup

### 1. Database (Docker, recommended)

```bash
docker compose up -d
```

This starts Postgres on `localhost:5432` with user `pm_user`, password
`pm_password`, database `pm_platform` -- matching `backend/.env.example`
exactly. (Already have Postgres installed locally instead? Skip this and
just make sure a `pm_platform` database exists, then point `DATABASE_URL`
at it.)

### 2. Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
# Edit .env: replace JWT_SECRET_KEY with a random value:
python -c "import secrets; print(secrets.token_hex(32))"
```

**Known dependency quirk:** `passlib==1.7.4` has a compatibility bug with
`bcrypt>=4.1` that throws `password cannot be longer than 72 bytes` even
for short passwords. `requirements.txt` already pins `bcrypt==4.0.1` to
avoid this -- if you ever bump bcrypt independently, re-pin it back down.

#### Database migration

```bash
alembic revision --autogenerate -m "initial schema"
alembic upgrade head
```

This generates one migration file describing all 13 tables and applies it.
Open the generated file in `alembic/versions/` once, just to see it -- it
should contain 13 `op.create_table(...)` calls.

#### Seed data (recommended -- makes the dashboard/board immediately useful)

```bash
python scripts/seed.py
```

Creates a workspace ("Acme Corp"), 4 users, 2 projects, 8 tasks across all
four statuses/priorities, subtasks, comments, activity, and notifications.
Safe to re-run -- it detects existing seed data and skips.

**Test accounts (all passwords: `password123`):**

| Email | Name | Role in "Acme Corp" |
|---|---|---|
| adil@example.com | Adil Khan | OWNER |
| sarah@example.com | Sarah Chen | ADMIN |
| marcus@example.com | Marcus Webb | MEMBER |
| priya@example.com | Priya Patel | MEMBER |

#### Run it

```bash
uvicorn app.main:app --reload
```

API docs (interactive): `http://localhost:8000/docs`

### 3. Frontend

```bash
cd frontend
npm install
cp .env.example .env   # defaults to http://localhost:8000
npm run dev
```

Open `http://localhost:5173`. Log in with a seeded account, or register a
fresh one and create your own workspace.

## Verifying the full flow

Log in as `adil@example.com`, then:

1. **Dashboard** -- real stats and charts for "Acme Corp" (not hardcoded).
2. **Projects -> Website Redesign** -- a Kanban board with real tasks.
3. **Drag a card** to another column -- refresh the page; it stays where
   you dropped it (persisted, not just a UI illusion).
4. **Click a task** -> assign it to a different user, add a subtask, post a
   comment (try `@sarah` to test mentions), upload a small file.
5. **Log in as `sarah@example.com` in a second browser** (or an incognito
   window) -- drag a card on Adil's session and watch it move on Sarah's
   board without a refresh (WebSocket broadcast). Comment as Sarah and
   watch it appear on Adil's open task modal.
6. **Notification bell** -- Sarah should see a live badge/update when
   mentioned or assigned, without reloading.
7. **Settings** (as Marcus, a MEMBER) -- try changing the workspace name;
   the field is disabled in the UI, and even editing the request directly
   in devtools gets a 403 from the API.
8. **My Tasks** -- shows only tasks assigned to the logged-in user, across
   every project in the workspace.

## API endpoint summary

```
POST   /auth/register
POST   /auth/login
GET    /users/me
PATCH  /users/me/password

POST   /workspaces                          GET /workspaces
GET    /workspaces/{id}                     PATCH /workspaces/{id}          DELETE /workspaces/{id}
GET    /workspaces/{id}/members             POST /workspaces/{id}/members
DELETE /workspaces/{id}/members/{member_id} PATCH /workspaces/{id}/members/{member_id}/role
GET    /workspaces/{id}/dashboard
GET    /workspaces/{id}/search?q=

POST   /workspaces/{id}/projects            GET /workspaces/{id}/projects
GET    /projects/{id}                       PATCH /projects/{id}            DELETE /projects/{id}
GET    /projects/{id}/members               POST /projects/{id}/members     DELETE /projects/{id}/members/{member_id}
GET    /projects/{id}/activities
GET    /projects/{id}/labels                POST /projects/{id}/labels

POST   /projects/{id}/tasks                 GET /projects/{id}/tasks  (filters: status, priority, assignee_id, label_id, due_date, search)
GET    /tasks/{id}                          PATCH /tasks/{id}               DELETE /tasks/{id}
POST   /tasks/{id}/subtasks                 PATCH /subtasks/{id}            DELETE /subtasks/{id}
POST   /tasks/{id}/comments                 GET /tasks/{id}/comments
PATCH  /comments/{id}                       DELETE /comments/{id}
POST   /tasks/{id}/attachments              GET /tasks/{id}/attachments
GET    /attachments/{id}/download           DELETE /attachments/{id}

GET    /notifications                       GET /notifications/unread-count
PATCH  /notifications/{id}/read             PATCH /notifications/read-all

WS     /ws/projects/{project_id}?token=...
WS     /ws/notifications?token=...
```

## Implemented features

Landing page - registration/login/logout with JWT - protected routes -
workspaces with OWNER/ADMIN/MEMBER roles - workspace member management -
projects - Kanban board (TODO/IN PROGRESS/REVIEW/DONE) with persisted
drag-and-drop - task priority/due date/labels/description - subtasks with
progress bars - comments with edit/delete-own and @mentions - activity
history per project - notifications (assignment, comments, mentions, added
to project/workspace, with live delivery + unread count) - real-time
WebSocket sync for tasks/subtasks/comments/attachments - backend-driven
global search (tasks/projects/people) - backend-driven filters on the
Kanban board - dedicated My Tasks page - Settings (profile, password
change, workspace admin) - file attachments (upload/download/delete,
type + size validated) - dashboard with real aggregate stats and charts -
seed script with documented test accounts.

## Known limitations

- **WebSockets are in-memory, single-process.** Fine for local dev / a
  single deployed instance; would need a pub/sub layer (at which point
  introducing Redis would be justified) to scale across multiple backend
  processes.
- **"My Tasks" fetches per-project and merges client-side** -- there's no
  single "all my tasks across the workspace" backend endpoint outside the
  dashboard's capped summary. Fine at the scale of a handful of projects;
  a dedicated `/workspaces/{id}/my-tasks` endpoint would be the next step
  if a workspace grows to dozens of projects.
- **Workspace invites require the invitee to already have an account** --
  there's no email-based "invite someone who hasn't signed up yet" flow.
- **No password reset / forgot-password flow** -- only authenticated
  password change (Settings -> Account).
- **File attachments are stored on local disk**, not swept up in database
  backups -- restoring a Postgres dump alone won't bring attachment bytes
  with it.
- **JWTs on the WebSocket URL as a query param** (browsers can't set custom
  headers on WebSocket handshakes) -- acceptable for local dev, but means
  the token can land in server access logs; a production deployment would
  want a short-lived, single-use connection ticket instead.

## Common troubleshooting

**`ModuleNotFoundError: No module named 'sqlalchemy'`** -- your venv isn't
active, or `pip install -r requirements.txt` was run outside it. Confirm
`(venv)` shows in your prompt, then re-run the install.

**`pydantic_core._pydantic_core.ValidationError: DATABASE_URL / JWT_SECRET_KEY Field required`**
-- `.env` doesn't exist yet, or you're not running the command from inside
`backend/` (pydantic-settings looks for `.env` in the current directory).

**`password cannot be longer than 72 bytes`** on register -- a
passlib/bcrypt version mismatch; see "Known dependency quirk" above.

**`connection to server ... failed: Connection refused`** -- Postgres
isn't running. `docker compose up -d`, or start your local Postgres
service.

**`sqlalchemy.exc.ProgrammingError` mentioning a missing table** -- you
haven't run `alembic upgrade head` yet, or you're pointed at a stale
database from before this schema existed (see the upgrade note at the top
of this file).

**Alembic: "Can't locate revision identified by ..."** -- your database's
`alembic_version` table references a migration file that isn't in this
codebase. See the upgrade note at the very top of this README.

**CORS errors in the browser console** -- check `CORS_ORIGINS` in
`backend/.env` includes `http://localhost:5173` exactly (no trailing
slash), and that the frontend's `VITE_API_BASE_URL` points at the
backend's actual port.

**WebSocket won't connect / no live updates** -- WebSocket URLs are built
from `VITE_API_BASE_URL` with `http` swapped for `ws` (see
`frontend/src/utils/ws.ts`); if you changed the backend's port, restart
the frontend dev server after updating `.env` so Vite picks up the change.
