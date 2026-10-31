# ETL Studio

A web-based, drag-and-drop **ETL pipeline builder** with a Rust backend and a
Rust/WASM frontend.

![ETL Studio](docs/screenshots/canvas.png)

> **Container image:** `ghcr.io/suryandi-wijaya/etl-rust` — public.
> This repository holds the public setup guide; the application source code is private.

## Why Rust?

- **No JVM or Node.js runtime.** The backend, scheduler, worker and UI all ship
  as native Rust/WASM artifacts, so the container only contains the app, not a
  large language runtime.
- **Small memory footprint.** The idle web process typically uses tens of MB of
  RSS (roughly 40–60 MB in a minimal single-container install); memory grows with
  active pipelines, connectors and dataset size rather than with a VM heap.
- **Predictable latency.** No garbage-collector pauses while executing a
  pipeline or streaming a large result.
- **Memory and thread safety.** The engine, API and permission checks are
  compiled with Rust's ownership guarantees; data races and use-after-free are
  rejected at compile time.
- **Fast startup.** The container becomes healthy in seconds, which keeps
  worker scale-up/scale-down responsive.
- **Compact architecture-specific image.** Around 440 MB uncompressed for
  both `linux/amd64` and `linux/arm64`.

## Setup

### 1. Download the image

```bash
# Intel/AMD64:
docker pull ghcr.io/suryandi-wijaya/etl-rust:latest-amd64

# Apple Silicon/ARM64:
docker pull ghcr.io/suryandi-wijaya/etl-rust:latest-arm64
```

If the pull is denied, the package may still be private or the visibility change
is still propagating — see the FAQ.

### 2. Create the `.env` file

In an empty folder, create `.env`:

```dotenv
ETL_SEED_ADMIN_PASSWORD=ChangeMe123
ETL_PG_PASSWORD=change-me-postgres
```

### 3. Create `docker-compose.yml`

Create `docker-compose.yml` in the same folder. It starts the UI/API, PostgreSQL
and the scheduler. Workers are intentionally **not** part of this file: they are
created from the UI so they can retire cleanly instead of being restarted by
Docker Compose.

```yaml
name: etl-studio

services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: etl
      POSTGRES_PASSWORD: ${ETL_PG_PASSWORD:-change-me-postgres}
      POSTGRES_DB: etl
    volumes:
      - etl-pg:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U etl -d etl"]
      interval: 10s
      timeout: 5s
      retries: 5

  etl-studio:
    image: ghcr.io/suryandi-wijaya/etl-rust:latest-amd64
    restart: unless-stopped
    ports:
      - "8787:8787"
    environment:
      ETL_SEED_ADMIN_PASSWORD: ${ETL_SEED_ADMIN_PASSWORD:-}
      ETL_EXECUTION_MODE: queue
      ETL_SCHEDULER_ENABLED: "0"
      ETL_ALLOW_MANAGED_WORKERS: "0"
      ETL_ALLOW_EXTERNAL_WORKER_CREATION: "1"
      ETL_NODE_RETRY_ATTEMPTS: "1"
      ETL_NODE_RETRY_DELAY_MS: "500"
      ETL_DATABASE_URL: "postgres://etl:${ETL_PG_PASSWORD:-change-me-postgres}@postgres:5432/etl"
      ETL_PG_SYNC_SECS: "5"
    volumes:
      - etl-data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      postgres:
        condition: service_healthy

  scheduler:
    image: ghcr.io/suryandi-wijaya/etl-rust:latest-amd64
    restart: unless-stopped
    depends_on:
      etl-studio:
        condition: service_healthy
    environment:
      ETL_ROLE: scheduler
      ETL_ROOT: /data
      ETL_EXECUTION_MODE: queue
      ETL_SCHEDULER_INTERVAL_SECS: "20"
    volumes:
      - etl-data:/data

volumes:
  etl-data:
  etl-pg:
```

> The `/var/run/docker.sock` mount lets the web container create and retire
> external workers from **System → Workers**. If you do not need that, remove the
> socket line and set `ETL_ALLOW_EXTERNAL_WORKER_CREATION: "0"`.

### 4. Start the stack

```bash
docker compose up -d
```

Check that everything is healthy:

```bash
docker compose ps
```

### 5. Sign in

Open <http://localhost:8787> and sign in with `admin` / `ChangeMe123` **on a
new, empty volume** (or `admin123` if `ETL_SEED_ADMIN_PASSWORD` was omitted).
If you have started this stack before, the existing volume keeps the old admin
password — use `admin` / `admin123` first, then change it from the account menu.

`ETL_SEED_ADMIN_PASSWORD` is only applied on the very first run when
`/data/auth.db` does not exist; changing `.env` later does not reset an existing
password.

Queued runs need a worker. Workers are not started by Compose (that would make
**Retire** restart them); create one from **System → Workers → New external
worker**. It uses the same image and shares the `etl-data` volume. You can now
create a pipeline in **Designer**, run it, and manage schedules, workers,
connections and environments from the UI.

---

## FAQ

<details>
<summary><strong>Can I run one container without Compose?</strong></summary>

Yes. macOS / Linux:

```bash
docker volume create etl-data
docker run -d --name etl-studio --restart unless-stopped \
  -p 8787:8787 -v etl-data:/data \
  ghcr.io/suryandi-wijaya/etl-rust:latest-amd64
```

Windows PowerShell:

```powershell
docker volume create etl-data
docker run -d --name etl-studio --restart unless-stopped `
  -p 8787:8787 -v etl-data:/data `
  ghcr.io/suryandi-wijaya/etl-rust:latest-amd64
```

Scheduled/queued runs need the scheduler service from the YAML above and a
worker created from **System → Workers → New external worker**.
</details>

<details>
<summary><strong>Why doesn't the password from `.env` work?</strong></summary>

`ETL_SEED_ADMIN_PASSWORD` is a **first-run seed only**. It is used when the
container creates `/data/auth.db` for the first time. If the `etl-data` volume
(or the PostgreSQL mirror in `etl-pg`) already exists, the password and users are
restored from there, so `.env` changes are ignored.

- Existing installation: sign in with the previous password (default
  `admin` / `admin123`) and change it from the account menu.
- Truly fresh start: `docker compose down -v` deletes all volumes, including
  pipelines and users, then `docker compose up -d` seeds `ChangeMe123` again.
- Never run `docker compose down -v` on an installation whose data you need.

</details>

<details>
<summary><strong>Where is my data and how do I back it up?</strong></summary>

Everything lives in the `etl-data` volume (`/data` in the container):
pipelines, runs, datasets, outputs, `auth.db`, `connections/` and `secret.key`.
Postgres stores the durable mirror in `etl-pg`.

Back up both volumes while the stack is stopped:

```bash
docker compose down
docker run --rm -v etl-studio_etl-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/etl-data-backup.tgz -C /data .
docker run --rm -v etl-studio_etl-pg:/data -v "$PWD":/backup alpine \
  tar czf /backup/etl-pg-backup.tgz -C /data .
docker compose up -d
```

Windows PowerShell: use `${PWD}:/backup` instead of `"$PWD":/backup`.
</details>

<details>
<summary><strong>What is secret.key / ETL_SECRET_KEY?</strong></summary>

Connections are encrypted with AES-256-GCM. On first start the server creates
`/data/secret.key`; it is included in the `etl-data` backup. Without it, saved
connections cannot be decrypted. You can supply a persistent base64 32-byte key
with `ETL_SECRET_KEY` instead. Never generate a new key for existing data.
</details>

<details>
<summary><strong>How do I update or roll back?</strong></summary>

```bash
docker compose pull
docker compose up -d
```

Tags are **architecture-specific**: `latest-amd64` or `latest-arm64`. The
compose example uses `latest-amd64`; on Apple Silicon change it to
`latest-arm64`. Set `ETL_IMAGE` in `.env` to keep the override in one place.
Data stays in the volumes.
</details>

<details>
<summary><strong>Apple Silicon / arm64?</strong></summary>

Use the arm64 tag on Apple Silicon:

```bash
docker pull ghcr.io/suryandi-wijaya/etl-rust:latest-arm64
```

In Compose set `ETL_IMAGE=ghcr.io/suryandi-wijaya/etl-rust:latest-arm64` (the
default is the amd64 tag). There is no multi-arch `latest` tag by design.
</details>

<details>
<summary><strong>The image pull is denied.</strong></summary>

The package may still be private or GitHub visibility is still propagating.
Authenticate with a classic token that has `read:packages`:

```bash
echo "$CR_PAT" | docker login ghcr.io -u suryandi-wijaya --password-stdin
```

Then retry `docker compose pull`.
</details>

<details>
<summary><strong>How do environments and deployments work?</strong></summary>

Use the environment switcher in the top bar to choose **Development**, **Test**,
**Production**, or a custom environment. Click the sliders button next to the
switcher to add or remove environments. Each environment has its own pipeline
documents and folder counts, so designs and versions are independent.

The three defaults are seeded on first start; **Development** is the protected
default. Custom environments can be removed once they contain no pipelines.

Open a pipeline and click **Deploy** in the Studio top bar to copy the current
design to another environment. The first deployment creates a copy; later
deployments update the matching pipeline in place (same pipeline key) and carry
the source version. The target card shows its environment and `vN`; every save
bumps the version of the pipeline you are editing.

Connections are scoped to environments. Deployment remaps a node's selected
connection by **name + kind** to the matching connection in the target
environment; if it does not exist yet the deployment fails and asks you to create
it first.

</details>

<details>
<summary><strong>How does automatic retry work?</strong></summary>

Failed nodes are retried automatically before the run fails. Configure it with:

- `ETL_NODE_RETRY_ATTEMPTS` — retries per node (default `1` in the Compose stack,
  `0` when unset outside Docker)
- `ETL_NODE_RETRY_DELAY_MS` — wait between attempts (default `500`)

Successful nodes are not re-executed; only the failed node is retried and the
run continues once it succeeds. Sinks are never retried to avoid duplicate
writes. Retries are recorded in the node log on the Steps tab.

A `Delay (test)` node is available in the palette to build long-running test
pipelines (for example 300000 ms = 5 minutes).
</details>

<details>
<summary><strong>How do I create a worker from the UI?</strong></summary>

Open **System → Workers → New external worker**. The web container creates a
Docker worker container that shares the workspace and network with the web
container; the new card appears as **External** after its first heartbeat.

This requires the Docker socket mount and
`ETL_ALLOW_EXTERNAL_WORKER_CREATION=1` (already set in the Compose file).
The worker uses the same image and `etl-worker` binary. Already-running workers
created this way can be edited from **System → Workers → Edit** (name,
concurrency, max rows, CPU/memory limits). Editing recreates the container with
the new limits, so an active job is interrupted. **Retire** stops the container
for good; because workers are not a Compose service, Docker will not restart
them.

> Mounting `/var/run/docker.sock` gives the web container host-level Docker
> access. Keep the UI behind HTTPS/trusted users, or disable
> `ETL_ALLOW_EXTERNAL_WORKER_CREATION` and start worker containers manually.
</details>

<details>
<summary><strong>How do I get scheduled runs and multiple workers?</strong></summary>

The `scheduler` service is in the YAML above. Create one or more workers from
**System → Workers → New external worker**; each worker runs with its own
concurrency. Workers exit when idle and are **not** restarted by Compose, so
**Retire** works as expected.

All pipeline runs (Studio **Run**, Operations and schedules) are queued for
workers; the web process never executes pipelines inline.
</details>

<details>
<summary><strong>How do I build from source or develop locally?</strong></summary>

Access to the private repository is required.

```bash
git clone https://github.com/suryandi-wijaya/etl-rust.git
cd etl-rust
make dev        # native web process: no Docker image build, runs inline
make dev-ui     # terminal 2: incremental Trunk UI on http://127.0.0.1:8080
make dev-docker # full Docker queue stack with scheduler and workers
```
</details>

<details>
<summary><strong>Security notes</strong></summary>

- Change the seeded `admin` password immediately.
- Put the app behind HTTPS/reverse proxy; do not expose port `8787` publicly.
- Keep `/data` and backups private; they contain credentials.
- The Docker socket mount gives the web container host-level Docker access; only
  enable it for trusted administrators, or remove it and run worker containers
  outside Compose.
- Runtime storage can be large; monitor disk usage on the System page.
</details>

<details>
<summary><strong>How are image releases published?</strong></summary>

Before triggering a build, update the **Changelog** section below with the new
version and replace the light-theme screenshot in `docs/screenshots/canvas.png`
if the UI changed.

Manually: Actions → **Publish GHCR (manual)** → Run workflow.

The matrix builds both architectures on native GitHub runners:
- **amd64:** `ubuntu-latest`
- **arm64:** `ubuntu-24.04-arm`

Each build publishes only one tag: `latest-amd64` or `latest-arm64`. Pushing to
`main` does not publish an image. Local Mac development uses `make dev` without
building a Docker image.
</details>

## Changelog

Every image release updates this section. Newest version first.

### 0.3.0 — 2026-10-29

- Connection edit now restores non-secret values; secrets remain write-only.
- Test connection is available from the connection card, create form, and edit form.
- GCP connectors support service-account JSON for GCS, BigQuery, and Pub/Sub.
- Untested connectors (S3, Redshift, Azure Blob, Snowflake, Kafka, RabbitMQ) are hidden until live-tested; MongoDB is enabled after Atlas smoke tests.
- Workers belong to pools; a Default pool is created automatically, pools have min/max workers (min 0 scales to zero), and the Workers tab is merged into Pools.
- PostgreSQL, Redshift, and MySQL sources can pick table names from metadata and
  use an optional soft sample limit.
- Operations adds richer run details, filters, queue metrics, failure alerts, and
  a compact execution-history grid.
- The data preview dock is vertically resizable; output columns render as a list.
- Pipelines can be renamed from the Studio top bar; both pipelines and chains can be renamed in Designer.
- Join can select output columns from both input tables for the next node.
- System and Operations screens are redesigned for daily operations.
- Docker builds reuse one image and Cargo/BuildKit caches for faster rebuilds.
- Both AMD64 and ARM64 images are built on GitHub Actions.
- File source nodes take a CSV/Excel/JSON file directly from the local computer or a mounted `/data/local` path; uploads stream to disk up to 10 GB and the global Data Library has been removed. Full runs have no application row cap. CSV → stateless-transform → CSV/JSON chains stream in bounded chunks; other pipelines still materialise rows in memory (see STREAMING.md in the app repo).
- GCP connectors accept either an access token or service-account JSON.
- Full database runs read all rows up to the configured maximum; sample rows is preview-only.
- UI-created workers remain standalone containers.
- Operations history shows run dates and times directly above the grid.
- Queue-mode Run lets the operator choose a worker or create one when none is online.
- Worker pools autoscale from zero with CPU, memory and disk limits for manual runs and schedules.
- Removed the `ETL_MAX_ROWS` environment parameter; the engine uses a fixed row guard.

### 0.2.0 — 2026-10-28

- Deploy console with per-asset diff and deploy log.
- Kafka and Pub/Sub micro-batch streaming with checkpoints, ack, and dedupe.
- Excel, XML, MongoDB, JSON flatten/unnest, text tools, and window nodes.
- HTTP pagination/auth connectors and a reorganized node catalog.
- System page split into Server, Workers, and Logs tabs.
- Workers are created from the UI and retire cleanly outside Compose.

### 0.1.0 — 2026-10-24

- Initial ETL Studio image: Rust/axum backend, Rust/WASM UI, pipeline canvas,
  transformations, connectors, RBAC, scheduler, and Docker deployment.

## License

MIT.

PostgreSQL nodes accept an optional libpq connection URI; Supabase direct database hosts may be IPv6-only, so use the project pooler URI on IPv4-only networks.

MySQL nodes accept an optional connection URI (including Aiven `ssl-mode=REQUIRED`); set a primary key for Create mode on Aiven clusters.

GCP BigQuery, GCS and Pub/Sub nodes accept service-account JSON/files, not only short-lived OAuth tokens. Live GCP tests are env-gated and skip when no service-account key is provided.

File nodes include clickable Browse file and Browse server pickers; Browse server lists workspace data and the read-only `/data/local` mount.

Operations has a read-only Designer-style live graph for running queue jobs, with rows-in → rows-out shown above every node; no Designer navigation is needed.

Admins can tail raw Docker container logs (web, scheduler, workers) from System → Logs; this requires the Docker socket and the users:manage permission.

CSV/JSONL/Parquet/Avro/MySQL sources and CSV/JSON/MySQL/Parquet sinks stream in bounded batches; Calculator, JSON Flatten, Unnest and Limit/Offset also run per chunk, so large datasets no longer need to fit in worker memory for those pipelines.

Live runs and recent run history show a STREAMING or BATCH execution-mode badge.

Workers advertise why they cannot accept claims (disk guard, memory guard, retiring); the run dialog skips paused workers, and a pinned job is reassigned if its target becomes paused.

Worker pool cards show live CPU, RAM and disk usage for each worker, along with any pause reason.

Worker discovery uses container naming conventions (`etl-studio-worker-*`), not Docker Compose project/service labels; override with `ETL_WORKER_NAME_PREFIX`.

The live run view tails recent node logs and the job-level error below the read-only pipeline graph.

Running jobs can be stopped from Operations; workers cancel between nodes/chunks and mark the job canceled.

Live streaming runs report per-chunk source/sink row movement in the Operations graph and current-node counters.

Worker cards in Operations can force-stop or restart a worker; force-stop fails its running jobs so they can be retried.

Streaming Limit/Offset stops reading the upstream source once the limit is reached, so a 100k limit does not load the full CSV.

File source nodes accept a Row limit option; streaming sources stop early and batch file readers truncate the result.

The Designer canvas has an Auto layout button that arranges nodes in layered columns and fits the result in view.

UI-created workers join the web container network by name, so they keep DNS after the web container is recreated.

The Operations and Live run views use a modern data-platform visual system with ambient surfaces, metric cards, a live graph and terminal-style logs.

Worker control is consolidated under System → Pools; Operations has no separate Workers tab and its Workers summary card opens Pools.
