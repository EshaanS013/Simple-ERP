# WOLF3D ERP

An internal ERP for WOLF3D Technologies covering work orders, projects, tasks,
quality control, dispatch, clients, notices, departments, reporting and team
management — with live realtime updates across all connected users.

Built to replace an Excel-based workflow and run on a company LAN for 10–50
concurrent users with a single command and **no client-side configuration**.

---

## Stack

- **Backend** — Node.js, Express, PostgreSQL (`pg`), Socket.IO, JWT auth, Zod validation, Helmet, rate limiting
- **Frontend** — React 18, Vite, Tailwind CSS, React Router, Axios, Socket.IO client
- **Delivery** — Docker Compose: PostgreSQL + backend + Nginx (reverse proxy & static host)

## Architecture

The browser only ever talks to **one origin** — the Nginx container. Nginx serves
the built React app and reverse-proxies `/api` and `/socket.io` to the backend, so
there are no hardcoded URLs and nothing to configure per client machine.

```
              ┌──────────────────────── company LAN ────────────────────────┐
  browser ──▶ │  web (nginx :80)  ──/api, /socket.io──▶  server (node :4000)  │
              │      static SPA                                  │            │
              │                                                  ▼            │
              │                                        db (postgres :5432)    │
              └──────────────────────────────────────────────────────────────┘
```

---

## Deploy on the LAN (recommended)

Requirements: Docker + Docker Compose on the host machine.

1. Configure environment:
   ```bash
   cp .env.example .env
   ```
   Edit `.env` and set at minimum:
   - `JWT_SECRET` — generate with `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"`
   - `POSTGRES_PASSWORD` — a strong database password
   - `ADMIN_PASSWORD` — first-login admin password (quote it if it contains `#`). Leave blank to have one generated and printed to the logs on first boot.

2. Launch:
   ```bash
   docker compose up -d --build
   ```

3. Find the host's LAN IP (`ip addr` / `ipconfig`) and open from any PC on the network:
   ```
   http://<LAN-IP>            # e.g. http://192.168.1.50
   ```
   Sign in as `admin` with the password from `.env`. You'll be prompted to set a
   new password on first login.

To change the published port, set `WEB_PORT` in `.env` (e.g. `WEB_PORT=8080`).

Useful operations:
```bash
docker compose logs -f server   # follow backend logs
docker compose ps               # service + health status
docker compose down             # stop (database volume is preserved)
docker compose down -v          # stop and delete all data
```

The database is stored in the named volume `db-data` and survives restarts and
rebuilds. Schema migrations and the admin bootstrap run automatically on startup.

---

## Local development

Requirements: Node.js 20+, a local PostgreSQL instance.

```bash
npm run install-all                 # install server + client deps

cp server/.env.example server/.env  # set DATABASE_URL, JWT_SECRET, ADMIN_PASSWORD
npm run dev                         # server :4000 + client :5173 (proxied)
```

The Vite dev server proxies `/api` and `/socket.io` to the backend, so the dev
experience matches production.

Run the test suites (backend must reach PostgreSQL):
```bash
cd server
node test_api.js      # REST: auth, CRUD, validation, RBAC  (38 checks)
node test_socket.js   # realtime: auth, presence, broadcast (4 checks)
```

---

## Roles & permissions

- **Manager** — full access: manages users, projects, work orders, clients,
  quality, dispatch, notices and department updates.
- **Employee** — reads operational data and can update the tasks assigned to them;
  cannot create projects, manage users, or edit unassigned tasks.

## Security highlights

- No default/hardcoded credentials — the admin is seeded from `ADMIN_PASSWORD`
  (or a generated one), and new users must change their password at first login.
- JWT secret is validated at boot; weak or short secrets are rejected in production.
- Per-route input validation, request rate limiting, login throttling, Helmet
  headers, and origin-restricted CORS and Socket.IO connections.
- Socket connections require a valid JWT; unauthenticated sockets are refused.
