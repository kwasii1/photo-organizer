# Photo Organizer

AI-powered photo organization — detect faces, cluster people, and manage photo projects with a real-time dashboard.

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌───────────────┐
│  PostgreSQL  │◄────│  FastAPI     │◄────│  Laravel UI   │
│  + pgvector  │     │  face-       │     │  (nginx/php/  │
│              │     │  pipeline    │     │  horizon/     │
└──────────────┘     └──────────────┘     │  reverb)      │
       │                    │             └───────────────┘
       │                    │                    │
       ▼                    ▼                    ▼
  postgres_data      shared_storage        Redis
                     (photos/crops)   (queues/websockets)
```

| Service                    | Image                                     | Port                    |
| -------------------------- | ----------------------------------------- | ----------------------- |
| PostgreSQL + pgvector      | `pgvector/pgvector:pg18`                  | `5432` (localhost only) |
| Redis                      | `redis:8-alpine`                          | `6379` (localhost only) |
| Face Pipeline (FastAPI)    | `ghcr.io/kwasii1/face-pipeline:latest`    | `8001` (internal)       |
| Face Pipeline UI (Laravel) | `ghcr.io/kwasii1/face-pipeline-ui:latest` | `80`, `8080`            |

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/)
- 8 GB+ RAM recommended (InsightFace models require ~4 GB)

## Quick Start

```bash
# Clone the repository
git clone https://github.com/kwasii1/photo-organizer.git
cd photo-organizer

# Create your environment file
cp .env.example .env

# Generate an application key
docker run --rm ghcr.io/kwasii1/face-pipeline-ui:latest php artisan key:generate --show

# Paste the output into your .env as APP_KEY, then generate Reverb secrets:
openssl rand -hex 16  # paste as REVERB_APP_KEY
openssl rand -hex 16  # paste as REVERB_APP_SECRET

# Start all services
docker compose up -d
```

Open `http://localhost` in your browser. The first startup may take a minute while the database schema is created and services warm up.

## Environment Variables

| Variable            | Default                     | Description                            |
| ------------------- | --------------------------- | -------------------------------------- |
| `DB_DATABASE`       | `face_pipeline`             | Database name                          |
| `DB_USERNAME`       | `postgres`                  | Database user                          |
| `DB_PASSWORD`       | `password`                  | Database password                      |
| `APP_KEY`           | —                           | Laravel encryption key (**required**)  |
| `APP_URL`           | `http://localhost`          | Public URL of the application          |
| `REVERB_APP_ID`     | `843193`                    | Reverb application ID                  |
| `REVERB_APP_KEY`    | —                           | Reverb WebSocket key (**required**)    |
| `REVERB_APP_SECRET` | —                           | Reverb WebSocket secret (**required**) |
| `FASTAPI_BASE_URL`  | `http://face-pipeline:8001` | Face detection service URL             |
| `RUN_MIGRATIONS`    | `true`                      | Auto-run database migrations on start  |
| `REDIS_PASSWORD`    | —                           | Redis password (leave empty for none)  |

> `DB_DATABASE`, `DB_USERNAME`, and `DB_PASSWORD` are shared across the Postgres container, the FastAPI face-pipeline, and the Laravel app — set them once in `.env` and all three services pick them up consistently.

## Useful Commands

```bash
# View logs
docker compose logs -f

# View logs for a specific service
docker compose logs -f app
docker compose logs -f face-pipeline

# Restart a service
docker compose restart app

# Stop everything
docker compose down

# Stop and delete all data (database + redis + storage)
docker compose down -v
```

## Multi-Container Deployment

The default compose file runs all Laravel processes (nginx, PHP-FPM, Horizon, Reverb) in a single container. For production deployments where you want to scale workers independently, use the multi-container variant:

```bash
docker compose -f docker-compose.multi.yml up -d
```

For the multi-container setup, make sure your `.env` contains the Reverb broadcast settings so the web container can reach the `reverb` service:

```dotenv
REVERB_BROADCAST_HOST=reverb
REVERB_BROADCAST_PORT=8080
REVERB_BROADCAST_SCHEME=http
```

If you copied `.env.example`, uncomment those lines or add them to `.env` before starting the stack.

| Service     | Container Role   | Port   | Notes                                      |
| ----------- | ---------------- | ------ | ------------------------------------------ |
| `web`       | nginx + PHP-FPM  | `80`   | Runs migrations on start                   |
| `horizon`   | Queue worker     | —      | Processes face detection / clustering jobs |
| `reverb`    | WebSocket server | `8080` | Real-time progress updates                 |
| `scheduler` | Cron scheduler   | —      | Runs Laravel's scheduled tasks             |

All four share the same image (`ghcr.io/kwasii1/face-pipeline-ui:latest`) with different `CONTAINER_ROLE` values. Infrastructure services (Postgres, Redis, face-pipeline) are identical to the all-in-one compose file and share the same named volumes.

Scale Horizon workers horizontally if you need to process more photos concurrently:

```bash
docker compose -f docker-compose.multi.yml up -d --scale horizon=3
```

## Persistent Data

All data is stored in Docker volumes:

| Volume           | Contents                                   |
| ---------------- | ------------------------------------------ |
| `postgres_data`  | Database (schema, faces, projects, people) |
| `redis_data`     | Queue jobs and Reverb state                |
| `shared_storage` | Uploaded photos and generated face crops   |

To back up, use `docker compose cp` or mount the volumes to backup targets.

## Upgrading

```bash
docker compose pull
docker compose up -d
```
