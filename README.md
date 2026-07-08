# Multi-Container Service (Todo API)

A Node.js/Express + MongoDB Todo API, orchestrated with Docker Compose across three containers: the API, MongoDB, and an NGINX reverse proxy. Deployed via a CI/CD pipeline that builds and pushes to GHCR, then deploys to a remote server over SSH.

## Project URL
https://roadmap.sh/projects/multi-container-service

## Stack
- Node.js 20 + Express + Mongoose
- MongoDB 7
- NGINX (reverse proxy)
- Docker Compose
- GitHub Container Registry (GHCR)
- Deployed to a Terraform-provisioned AWS EC2 instance (see [devops-projects](https://github.com/WilliamFly/devops-projects/tree/main/19-multi-container-service))

## Endpoints

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/todos` | Get all todos |
| POST | `/todos` | Create a new todo |
| GET | `/todos/:id` | Get a single todo by id |
| PUT | `/todos/:id` | Update a single todo by id |
| DELETE | `/todos/:id` | Delete a single todo by id |

## Architecture

Three containers on a shared internal Docker network (`todo-network`):

- **`mongo`** — no exposed ports; only reachable by other containers on the network. Data persisted via a named volume (`mongo-data`), so it survives container restarts and even full `docker compose down`/`up` cycles.
- **`api`** — no exposed ports either. Reachable internally at `mongo:27017` and `api:3000` by service name (Docker Compose's built-in DNS resolves service names automatically).
- **`nginx`** — the *only* container exposing a port to the host (`80`). Reverse-proxies all traffic to `api:3000` internally.

```
Internet → NGINX (port 80) → api:3000 (internal only) → mongo:27017 (internal only)
```

## Environment Variables

Copy `.env.example` to `.env`:

```
PORT=3000
MONGO_URI=mongodb://mongo:27017/tododb
NODE_ENV=development
```

## Local Development

```bash
docker compose up --build -d
curl http://localhost/todos
```

Note: requests go through NGINX on port 80, not directly to the API on port 3000 — the API is intentionally not exposed to the host.

## CI/CD Pipeline

On every push to `main`, `.github/workflows/deploy.yml`:

1. **build-and-push** — builds the API's Docker image and pushes it to `ghcr.io/williamfly/multi-container-service`
2. **deploy** — SSHs into the remote server, pulls the latest code (for compose/nginx config changes) and the latest image, then runs `docker compose pull && docker compose up -d` — no image is ever rebuilt on the production server itself

## Required GitHub Secrets

| Secret | Purpose |
|--------|---------|
| `SERVER_IP` | Public IP of the deployment target |
| `SSH_PRIVATE_KEY` | Key used to SSH into the server for deployment |
