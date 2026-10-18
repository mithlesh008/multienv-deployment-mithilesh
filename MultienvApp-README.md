# MultiEnv Ticket Management System

A Dockerized full-stack ticket management application with separate development and production backend environments.

Repository: <https://github.com/mithlesh008/multienv-deployment-mithilesh>

## Overview

This project demonstrates how to run multiple application environments from one Docker Compose configuration:

* A React frontend served on port `3000`
* A development Flask backend served on port `3001`
* A production Flask backend served on port `3002`
* A dedicated MongoDB instance for development data
* A dedicated MongoDB instance for production data

The development and production environments use separate backend containers and separate MongoDB volumes to keep their data isolated.

## Architecture

```text
                         +----------------------+
                         |   React Frontend     |
                         |   http://localhost:3000
                         +----------+-----------+
                                    |
                    +---------------+---------------+
                    |                               |
                    v                               v
       +--------------------------+    +--------------------------+
       | Development Backend     |    | Production Backend      |
       | Flask - port 3001       |    | Gunicorn - port 3002    |
       +------------+-------------+    +------------+-------------+
                    |                               |
                    v                               v
       +--------------------------+    +--------------------------+
       | Development MongoDB     |    | Production MongoDB      |
       | mongo-dev               |    | mongo-prod               |
       +--------------------------+    +--------------------------+
```

All services communicate over the private Docker network named `multienv`.

## Project Structure

```text
.
├── docker-compose.yaml
├── backend/
│   ├── dev/
│   │   ├── app.py
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   └── prod/
│       ├── app.py
│       ├── Dockerfile
│       └── requirements.txt
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   ├── public/
│   └── src/
├── .gitignore
└── README.md
```

## Services and Ports

| Service | Container port | Host port | Purpose |
|---|---:|---:|---|
| `frontend` | `3000` | `3000` | React user interface |
| `dev-backend` | `3001` | `3001` | Development API |
| `prod-backend` | `3002` | `3002` | Production API |
| `mongo-dev` | `27017` | Not exposed | Development database |
| `mongo-prod` | `27017` | Not exposed | Production database |

MongoDB is intentionally not exposed directly to the host. The backend containers connect to MongoDB through Docker's internal network.

## Prerequisites

Install the following tools before running the project:

* Docker Desktop, or Docker Engine with the Compose plugin
* Git
* A GitHub account with access to the repository

Verify Docker is available:

```bash
docker --version
docker compose version
```

## Configuration

The Compose configuration uses the following backend environment variables:

```text
PORT=3001
MONGO_URI=mongodb://mongo-dev:27017/tickets_dev
```

The production backend uses:

```text
PORT=3002
MONGO_URI=mongodb://mongo-prod:27017/tickets_prod
```

Do not commit real `.env` files, credentials, API keys, or production secrets. Use `.env.example` files for safe configuration templates.

## Run the Application

Clone the repository:

```bash
git clone https://github.com/mithlesh008/multienv-deployment-mithilesh.git
cd multienv-deployment-mithilesh
```

Build and start all services:

```bash
docker compose -f docker-compose.yaml up -d --build
```

Check the running containers:

```bash
docker compose -f docker-compose.yaml ps
docker ps
```

Open the application:

* Frontend: <http://localhost:3000>
* Development environment: <http://localhost:3001/dev>
* Production environment: <http://localhost:3002/prod>

## Stop the Application

Stop the containers without deleting database volumes:

```bash
docker compose -f docker-compose.yaml down
```

Stop the containers and remove the database volumes:

```bash
docker compose -f docker-compose.yaml down -v
```

The `-v` option permanently deletes the development and production MongoDB data stored in Docker volumes.

## Useful Docker Commands

View logs for all services:

```bash
docker compose -f docker-compose.yaml logs -f
```

View frontend logs:

```bash
docker compose -f docker-compose.yaml logs -f frontend
```

View backend logs:

```bash
docker compose -f docker-compose.yaml logs -f dev-backend prod-backend
```

Restart one service:

```bash
docker compose -f docker-compose.yaml restart frontend
```

Rebuild only the frontend:

```bash
docker compose -f docker-compose.yaml build --no-cache frontend
docker compose -f docker-compose.yaml up -d frontend
```

Validate the Compose file before starting:

```bash
docker compose -f docker-compose.yaml config
```

## Frontend Build Details

The frontend uses Create React App and is built in a multi-stage Dockerfile.

The build stage:

* Uses Node.js 20 Alpine
* Installs npm dependencies
* Installs compatible AJV packages required by `react-scripts` 5
* Generates the production build

The runtime stage:

* Uses Node.js 20 Alpine
* Installs the lightweight `serve` package
* Serves the generated React build on port `3000`

The Dockerfile includes the following compatibility settings:

```dockerfile
ENV CI=false
ENV NODE_OPTIONS=--openssl-legacy-provider
```

The AJV dependency versions are pinned for compatibility with the Create React App toolchain:

```dockerfile
RUN npm install --no-save --legacy-peer-deps \\
    ajv@6.12.6 \\
    ajv-keywords@3.5.2
```

## Troubleshooting

### Frontend build fails with `Cannot find module 'ajv/dist/compile/codegen'`

This indicates an incompatible `ajv` and `ajv-keywords` dependency combination. Confirm that the frontend Dockerfile installs:

```dockerfile
ajv@6.12.6
ajv-keywords@3.5.2
```

Then rebuild without using the Docker cache:

```bash
docker compose -f docker-compose.yaml build --no-cache frontend
docker compose -f docker-compose.yaml up -d frontend
```

### Show the complete frontend build output

Use plain progress output to display the full npm error:

```bash
docker compose -f docker-compose.yaml build \\
  --no-cache \\
  --progress=plain \\
  frontend
```

### Port already in use

Check which process is using a port:

```bash
# Linux or macOS
lsof -i :3000
lsof -i :3001
lsof -i :3002

# Windows PowerShell
Get-NetTCPConnection -LocalPort 3000,3001,3002
```

Stop the conflicting process or change the host-side port mapping in `docker-compose.yaml`.

### Container starts but the application is unavailable

Check service status and logs:

```bash
docker compose -f docker-compose.yaml ps
docker compose -f docker-compose.yaml logs --tail=100 frontend
docker compose -f docker-compose.yaml logs --tail=100 dev-backend
docker compose -f docker-compose.yaml logs --tail=100 prod-backend
```

### Reset the application completely

The following command removes containers, networks, and database volumes:

```bash
docker compose -f docker-compose.yaml down -v --remove-orphans
docker compose -f docker-compose.yaml up -d --build
```

Use this only when it is safe to delete local database data.

## Verification Checklist

After deployment, verify the following:

* `docker compose ps` shows all five services running
* The frontend responds at `http://localhost:3000`
* The development backend responds on port `3001`
* The production backend responds on port `3002`
* Development and production use different MongoDB services
* No `.env` files or secrets are committed
* The frontend image builds successfully without cache

Basic frontend check:

```bash
curl -I http://localhost:3000
```

Expected result:

```text
HTTP/1.1 200 OK
```

## GitHub Submission Workflow

Check the working tree:

```bash
git status
git diff --check
```

Review changed files:

```bash
git diff --stat
git diff
```

Stage and commit the assignment:

```bash
git add -A
git commit -m "Deploy multi-environment ticket management app"
```

Push to the `main` branch:

```bash
git branch -M main
git push -u origin main
```

Confirm the latest commit:

```bash
git log -1 --oneline
git status
```

## Security Notes

* Never commit `.env` files or credentials.
* Do not expose MongoDB ports unless direct database access is required.
* Use separate databases and volumes for development and production.
* Use strong, environment-specific secrets outside source control.
* Review Docker logs before sharing them because they may contain configuration values.
