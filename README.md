# blog-node-angular

A minimal dockerized full-stack blog application demonstrating the **decoupled SPA + API** pattern.

- **Frontend:** Angular (served by nginx in production)
- **Backend:** Node.js + Express
- **Database:** MongoDB
- **Orchestration:** Docker Compose

The application exposes a small REST API for creating and listing blog posts, backed by a MongoDB database, with an Angular single-page application consuming that API from the browser.

---

## Architecture

Three services run in isolated containers on a shared custom Docker network:

| Service    | Image / Build         | Purpose                          | Published port |
| ---------- | --------------------- | -------------------------------- | -------------- |
| `frontend` | `./frontend` (nginx)  | Serves the compiled Angular SPA  | `4200 → 8080`  |
| `backend`  | `./backend` (Node.js) | REST API (`/health`, `/posts`)   | `3000 → 3000`  |
| `db`       | `mongo:7`             | Persistent document database     | *not exposed*  |

The database is intentionally not published to the host — it is only reachable from other containers on the internal `appnet` network. Data is persisted to a named Docker volume (`db-data`) so it survives container restarts.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) 20.10 or newer
- Docker Compose v2 (bundled with Docker Desktop)

That is the entire dependency list. Node.js, MongoDB, and the Angular CLI are all inside the containers — nothing needs to be installed on your host machine.

---

## Getting started

````bash
# 1. Clone the repository
git clone https://github.com/<your-username>/blog-node-angular.git
cd blog-node-angular

# 2. Create a local .env from the example
cp .env.example .env
# (on Windows: copy .env.example .env)
# Edit .env and set a strong MONGO_PASSWORD

# 3. Bring the whole stack up
docker compose up --build
````

The first build takes a few minutes while base images are pulled and dependencies installed. Subsequent starts are much faster thanks to Docker layer caching.

Once all three services are running:

| URL                                                          | What you'll see                                   |
| ------------------------------------------------------------ | ------------------------------------------------- |
| [http://localhost:4200](http://localhost:4200)               | Angular frontend                                  |
| [http://localhost:3000](http://localhost:3000)               | Backend root — plain text confirmation            |
| [http://localhost:3000/health](http://localhost:3000/health) | JSON health check (used by Docker's healthcheck)  |
| [http://localhost:3000/posts](http://localhost:3000/posts)   | JSON array of blog posts                          |

To stop the stack (keeping data): press `Ctrl+C`, then `docker compose down`.
To stop **and wipe the database**: `docker compose down -v`.

---

## API reference

### `GET /health`

Returns the health status of the API and its database connection.

````json
{ "status": "ok", "database": "connected" }
````

### `GET /posts`

Returns all posts, most recent first.

````json
[
  {
    "_id": "6533...",
    "title": "First post",
    "content": "Hello world",
    "createdAt": "2026-09-05T12:34:56.000Z"
  }
]
````

### `POST /posts`

Creates a new post. Both fields are required.

````bash
curl -X POST http://localhost:3000/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"My post","content":"Some content"}'
````

Returns the created post with HTTP status `201`. Missing fields return HTTP `400`.

---

## Configuration

All configuration lives in `.env` at the project root. The file is git-ignored; a template with placeholder values is committed as `.env.example`.

| Variable         | Purpose                        |
| ---------------- | ------------------------------ |
| `MONGO_USER`     | Root username for MongoDB      |
| `MONGO_PASSWORD` | Root password for MongoDB      |
| `MONGO_DB`       | Initial database name          |

Never commit a real `.env`. If you accidentally push one, rotate the credentials immediately.

---

## Docker design notes

- **Multi-stage builds.** Both the frontend and backend Dockerfiles use multi-stage builds. The Angular image is around 50 MB (a slim nginx serving only compiled static files); the backend runtime image ships only production dependencies.
- **Non-root users.** The frontend runs as `nginx` in `nginx-unprivileged` on port `8080`; the backend runs as an unprivileged `nodejs` user.
- **Layer caching.** Dependency manifests (`package.json`, `package-lock.json`) are copied and installed before the rest of the source, so source changes don't invalidate the dependency install layer.
- **Pinned base images.** No `:latest` tags — reproducible builds.
- **Healthchecks and startup ordering.** MongoDB has a `mongosh` ping healthcheck; the backend uses `depends_on: condition: service_healthy` so it doesn't attempt to connect before MongoDB accepts connections.
- **Named volume.** MongoDB data lives in the named volume `db-data` and survives `docker compose down`. Only `docker compose down -v` wipes it.
- **Isolated network.** All services communicate over a declared custom bridge network (`appnet`). The database is not published to the host.
- **`.dockerignore` per image.** Local `node_modules`, `.git`, `.env`, and build outputs are excluded from Docker build contexts.

---

## Project layout

````
blog-node-angular/
├── frontend/                # Angular application
│   ├── Dockerfile           # multi-stage: Node build → nginx runtime
│   ├── .dockerignore
│   ├── nginx.conf           # SPA-friendly nginx config
│   └── ...                  # Angular source
├── backend/                 # Express API
│   ├── Dockerfile           # multi-stage: deps → runtime
│   ├── .dockerignore
│   ├── index.js             # Server + routes
│   └── package.json
├── docker-compose.yml       # Orchestrates all three services
├── .env.example             # Template — safe to commit
├── .env                     # Real values — git-ignored
├── .gitignore
└── README.md
````

---

## Troubleshooting

**Backend logs show `MongoDB connection error`.** MongoDB probably wasn't fully ready. The healthcheck normally handles this — if it still fails, try `docker compose down` and `docker compose up` again.

**Port 3000 or 4200 already in use.** Something else on your machine is using that port. Either stop it, or edit `docker-compose.yml` and change the host side of the mapping (e.g. `"3001:3000"`).

**Changes to source aren't reflected.** The images are built once. Rebuild with `docker compose up --build`.

**Want to inspect the database.** Run `docker compose exec db mongosh -u $MONGO_USER -p $MONGO_PASSWORD --authenticationDatabase admin`.