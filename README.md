# Smart Cobot Factory Management System — MERN

Role-based factory management (Admin / Manager / Employee) for collaborative robots,
production jobs, work assignments, safety alerts, maintenance and real-time notifications.

| Layer | Technology |
|-------|------------|
| **M**ongoDB | Mongoose 8 models (14 collections) |
| **E**xpress | Express 5 REST API, JWT in httpOnly cookie, bcrypt, role + ownership guards |
| **R**eact | React 19 + Vite + TypeScript, Tailwind v4, shadcn/ui, Recharts, Zustand |
| **N**ode | Node 18+ · Socket.IO real-time · background jobs (cobot simulator, schedule dispatcher, workload snapshots) |

## Quick start

Requirements: **Node 18+** and a **MongoDB** instance (local, Docker or Atlas).

```bash
# 1. MongoDB (skip if you already have one running / use Atlas)
docker compose up -d

# 2. install everything + create server/.env
npm run setup

# 3. load demo data
npm run seed

# 4. run API (http://localhost:5000) + React dev server (http://localhost:5173)
npm run dev
```

Open **http://localhost:5173**.

### Demo accounts

| Role | Email | Password |
|------|-------|----------|
| Admin | `admin@factory.com` | `admin123` |
| Manager | `john.manager@factory.com` | `manager123` |
| Employee | `arun.emp@factory.com` | `employee123` |

### Production (single process)

```bash
npm run build      # builds client/dist
npm start          # Express serves API + the built React app on :5000
```

## Configuration (`server/.env`)

| Variable | Default | Notes |
|----------|---------|-------|
| `PORT` | `5000` | API port |
| `MONGODB_URI` | `mongodb://127.0.0.1:27017/smart_cobot_factory` | Local or Atlas `mongodb+srv://…` |
| `JWT_SECRET` | *(dev default)* | **Change in production** |
| `JWT_EXPIRES_IN` | `7d` | |
| `CLIENT_URL` | `http://localhost:5173` | CORS origin (only for split hosting) |
| `ENABLE_JOBS` | `true` | `false` disables the background jobs |

## Project structure

```
smart-cobot-mern/
├── package.json              # root scripts (setup, dev, build, start, seed)
├── docker-compose.yml        # optional local MongoDB
├── client/                   # ── React (Vite + TS) ──
│   ├── index.html
│   ├── vite.config.ts        # dev proxy: /api and /socket.io → :5000
│   └── src/
│       ├── main.tsx · App.tsx
│       ├── components/
│       │   ├── login/ · app-shell.tsx · command-palette.tsx
│       │   ├── dashboards/{admin,manager,employee,shared}/
│       │   └── ui/           # shadcn/ui primitives
│       ├── store/            # Zustand: auth, router
│       ├── hooks/            # use-socket, use-toast, use-mobile
│       └── lib/              # api client, types, utils
└── server/                   # ── Node + Express + MongoDB ──
    └── src/
        ├── index.js          # boot: DB → HTTP → Socket.IO → jobs
        ├── app.js            # Express app (CORS, cookies, routes, static client)
        ├── config/           # env, mongoose connection
        ├── models/           # User, Manager, Employee, Cobot, ProductionJob, Assignment,
        │                     # Notification, SafetyAlert, MaintenanceLog, ProductionSchedule,
        │                     # Setting, AuditLog, WorkloadSnapshot, CobotUtilizationHistory
        ├── middleware/       # auth (JWT, protect(roles), loadManager/loadEmployee), errors
        ├── controllers/      # auth, profile, manager, employee, admin/*
        ├── routes/           # auth, admin, manager, employee
        ├── jobs/             # cobotSimulator (5s), scheduleDispatcher (30s), workloadSnapshot (5m)
        ├── utils/            # socket hub, notify, audit, stats, cascade deletes, helpers
        └── seed/seed.js      # demo data
```

## Security model

* Passwords hashed with **bcrypt**; sessions are a **JWT in an httpOnly, SameSite=Lax cookie**.
* `protect("ADMIN", …)` guards every route; the React router mirrors it on the client.
* **Ownership checks are server-side**: a manager can only view/assign to employees whose `manager`
  is that manager; an employee can only start/complete their own assignments.
* Deactivated accounts cannot sign in; every important action is written to the **audit log**.

## Real-time (Socket.IO)

The Socket.IO server lives inside the Express process (no separate services).
Client emits `identify { userId }`; the server pushes `notification` (per user) and
`cobot-update`, `kpi-update`, `schedule-dispatched` (broadcast). Notification preferences in
**Settings** decide whether a live push is sent (the DB notification is always stored).

## REST API (all under `/api`)

* **Auth**: `POST /auth/login` · `GET /auth/me` · `POST /auth/logout` · `GET|PUT /profile`
* **Admin** (`/admin/*`): dashboard, analytics, performance, managers, employees (+ timeline, workload-history),
  cobots (+ detail, firmware), factory-map, jobs, safety, maintenance, predictive-maintenance, notifications,
  schedules, audit (+ export csv/pdf), reports (csv/json, pdf), search, settings, workload-snapshots
* **Manager** (`/manager/*`): dashboard, employees, jobs, cobots, team-productivity, assign-work, assignments, notifications
* **Employee** (`/employee/*`): dashboard, my-work (+ `:id/start`, `:id/complete`), notifications

## Notes

* IDs are MongoDB ObjectIds serialized as `id` strings; human IDs (`MGR001`, `EMP001`, `Cobot-01`, `JOB101`) are kept.
* Background jobs run in-process. To scale horizontally run them on a single instance (`ENABLE_JOBS=false` elsewhere).
* Reports "PDF" opens a print-ready page (use the browser's *Save as PDF*).
