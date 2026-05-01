# Local Development Setup

This document describes how the local development environment for Taipei City Dashboard was set up.

## Architecture

- **Frontend** (Vue/Vite) — runs locally on port 3000
- **Backend** (Go/Gin) — runs locally on port 8080
- **Databases** (PostgreSQL x2, Redis) — run in Docker containers

## Prerequisites Installed

- Node.js 18, npm
- Go 1.22
- Docker (docker-compose plugin)

## Docker Containers

Three containers are managed via `docker/docker-compose-db.yaml`:

| Container | Image | Host Port |
|---|---|---|
| `postgres-data` | postgis/postgis:16-3.4-alpine | 5433 |
| `postgres-manager` | postgis/postgis:16-3.4-alpine | 5432 |
| `redis` | redis:7.2.3-alpine | 6379 |

Two port mappings were added to `docker/docker-compose-db.yaml` so the locally running backend can reach the containers:
- `postgres-data`: `5433:5432`
- `redis`: `6379:6379`

(`postgres-manager` already had `5432:5432`.)

A Docker network must exist before starting the containers:

```bash
sudo docker network create br_dashboard
```

Start the containers:

```bash
cd docker
sudo docker compose -f docker-compose-db.yaml up -d redis postgres-data postgres-manager
```

## Database Initialization

A symlink is required because the backend hardcodes `/opt/db-sample-data/` as the sample data directory:

```bash
sudo mkdir -p /opt/db-sample-data
sudo ln -sf $(pwd)/db-sample-data/dashboard-demo.sql /opt/db-sample-data/dashboard-demo.sql
sudo ln -sf $(pwd)/db-sample-data/dashboardmanager-demo.sql /opt/db-sample-data/dashboardmanager-demo.sql
```

Run the following **once** to migrate the manager DB schema and seed both databases:

```bash
cd Taipei-City-Dashboard-BE

# 1. Migrate manager DB schema and create admin user
DB_MANAGER_HOST=localhost DB_MANAGER_PORT=5432 DB_MANAGER_USER=postgres \
DB_MANAGER_PASSWORD=postgres DB_MANAGER_DBNAME=dashboardmanager \
DB_DASHBOARD_HOST=localhost DB_DASHBOARD_PORT=5433 DB_DASHBOARD_USER=postgres \
DB_DASHBOARD_PASSWORD=postgres DB_DASHBOARD_DBNAME=dashboard \
DASHBOARD_DEFAULT_USERNAME=admin DASHBOARD_DEFAULT_Email=admin@example.com \
DASHBOARD_DEFAULT_PASSWORD=Admin123! \
go run main.go migrateDB

# 2. Load sample data into manager DB
sudo docker exec -i postgres-manager psql -U postgres -d dashboardmanager \
  < ../db-sample-data/dashboardmanager-demo.sql

# 3. Initialize and seed the dashboard DB
DB_MANAGER_HOST=localhost DB_MANAGER_PORT=5432 DB_MANAGER_USER=postgres \
DB_MANAGER_PASSWORD=postgres DB_MANAGER_DBNAME=dashboardmanager \
DB_DASHBOARD_HOST=localhost DB_DASHBOARD_PORT=5433 DB_DASHBOARD_USER=postgres \
DB_DASHBOARD_PASSWORD=postgres DB_DASHBOARD_DBNAME=dashboard \
PGPASSWORD=postgres \
go run main.go initDashboard
```

Default admin credentials: `admin` / `Admin123!`

## Running the Backend

```bash
cd Taipei-City-Dashboard-BE

export DB_MANAGER_HOST=localhost DB_MANAGER_PORT=5432 DB_MANAGER_USER=postgres
export DB_MANAGER_PASSWORD=postgres DB_MANAGER_DBNAME=dashboardmanager
export DB_DASHBOARD_HOST=localhost DB_DASHBOARD_PORT=5433 DB_DASHBOARD_USER=postgres
export DB_DASHBOARD_PASSWORD=postgres DB_DASHBOARD_DBNAME=dashboard
export REDIS_HOST=localhost REDIS_PORT=6379 REDIS_DB=0
export GIN_DOMAIN=0.0.0.0 GIN_PORT=8080 GIN_MODE=debug
export JWT_SECRET=secret IDNO_SALT=salt

go run main.go
```

The API is served at `http://localhost:8080/api/v1/`.

### Note on AI Features

The backend depends on the ONNX Runtime shared library (`/usr/lib/libonnxruntime.so`) for vector search features. This library is not available in the standard Ubuntu apt repositories. Two lines in `app/models/qdrant.go` were changed from `log.Fatalf` to `log.Printf` so the server starts gracefully without it — all dashboard and auth features work normally, only AI/vector-search endpoints are disabled.

## Running the Frontend

A `LOCAL_BACKEND` mode was added to `vite.config.js`. When `LOCAL_BACKEND=true`, Vite proxies `/api/*` to `http://localhost:8080/v1/*` instead of the production API.

```bash
cd Taipei-City-Dashboard-FE
npm install
LOCAL_BACKEND=true npm run dev
```

The app is served at `http://localhost:3000/` (bound to `0.0.0.0`, so also reachable via LAN IP).

## tmux Session

All three processes run in a tmux session named `dashboard`:

```bash
tmux attach -t dashboard
```

| Window | Content |
|---|---|
| `backend` | Go server |
| `frontend` | Vite dev server |
| `dblogs` | Docker container logs |

To recreate the tmux session from scratch after a reboot:

```bash
# 1. Restart Docker containers
cd /home/winlab/Taipei-City-Dashboard/docker
sudo docker compose -f docker-compose-db.yaml up -d redis postgres-data postgres-manager

# 2. Start tmux session
tmux new-session -d -s dashboard -n backend
tmux send-keys -t dashboard:backend "cd /home/winlab/Taipei-City-Dashboard/Taipei-City-Dashboard-BE && \
  export DB_MANAGER_HOST=localhost DB_MANAGER_PORT=5432 DB_MANAGER_USER=postgres \
  DB_MANAGER_PASSWORD=postgres DB_MANAGER_DBNAME=dashboardmanager \
  DB_DASHBOARD_HOST=localhost DB_DASHBOARD_PORT=5433 DB_DASHBOARD_USER=postgres \
  DB_DASHBOARD_PASSWORD=postgres DB_DASHBOARD_DBNAME=dashboard \
  REDIS_HOST=localhost REDIS_PORT=6379 REDIS_DB=0 \
  GIN_DOMAIN=0.0.0.0 GIN_PORT=8080 GIN_MODE=debug \
  JWT_SECRET=secret IDNO_SALT=salt && go run main.go" Enter

tmux new-window -t dashboard -n frontend
tmux send-keys -t dashboard:frontend "cd /home/winlab/Taipei-City-Dashboard/Taipei-City-Dashboard-FE && LOCAL_BACKEND=true npm run dev" Enter

tmux new-window -t dashboard -n dblogs
tmux send-keys -t dashboard:dblogs "sudo docker compose -f /home/winlab/Taipei-City-Dashboard/docker/docker-compose-db.yaml logs -f" Enter
```
