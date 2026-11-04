# ferrous-docker

[← Back to FerrousCX Root Architecture](https://github.com/reidlai/ferrous-cx)

> **Unified Multi-Container Runtime Orchestration & OpenTelemetry Telemetry for FerrousCX**  
> Complete local developer orchestration harness managing all 12 platform submodules via Docker Compose with explicit dependency graphs, healthcheck gating, pre-commit automation, and 3-tier OpenTelemetry developer runtime.

[![Runtime: Docker Compose](https://img.shields.io/badge/orchestration-Docker%20Compose%20v2.29+-blue.svg)](https://docs.docker.com/compose/)
[![Telemetry: OpenTelemetry](https://img.shields.io/badge/telemetry-OpenTelemetry%20%2B%20Jaeger-orange.svg)](https://opentelemetry.io/)
[![Architecture: 12 Submodules](https://img.shields.io/badge/topology-12%20Platform%20Submodules-purple.svg)](#12-submodule-topology)
[![DevSecOps: Multi-Submodule Gate](https://img.shields.io/badge/DevSecOps-Pre--Commit%20%2B%20Smoke%20Tests-brightgreen.svg)](#devsecops--smoke-testing)

---

## 🎯 Architectural Role

`ferrous-docker` provides the unified integration and developer runtime harness for the entire FerrousCX platform:

1. **Zero-Daemon Local Development:** Launches all 12 platform services with a single `docker compose up -d` command, requiring zero external host dependencies.
2. **Explicit Healthcheck Dependency Graph:** Orchestrates startup sequence using `condition: service_healthy` gating (e.g. `ferrous-iam` and `ferrous-streaming` boot first, followed by storage, API gateway, reverse proxy, and frontend).
3. **3-Tier OpenTelemetry Developer Runtime:**
   - **Tier 1 (Disabled):** `OTEL_SDK_DISABLED=true` (zero span overhead, clean stdout).
   - **Tier 2 (Console):** `OTEL_TRACES_EXPORTER=console` (human-readable trace logs in terminal).
   - **Tier 3 (OTLP Egress):** Ephemeral in-memory Jaeger All-In-One container (`jaegertracing/all-in-one:latest`) on ports 4317/4318 with Web UI on port 16686.
4. **Pre-Commit DevSecOps Automation:** Centralized hooks verifying secret leakage (`gitleaks`), Dockerfile hygiene (`hadolint`), and vulnerability scanning (`trivy`) across all submodules.
5. **Multi-Submodule Smoke Validation:** Scripted end-to-end integration test suites (`smoke-test.sh` / `smoke-test.ps1`) executing end-to-end multi-submodule lifecycle checks.

```text
                               ┌──────────────────────────┐
                               │  ferrous-reverseproxy    │
                               │  (Zentinel Edge Proxy)   │
                               │        :443 / :80        │
                               └────────────┬─────────────┘
                                            │
                     ┌──────────────────────┴──────────────────────┐
                     ▼                                             ▼
       ┌──────────────────────────┐                  ┌──────────────────────────┐
       │     ferrous-frontend     │                  │       ferrous-api        │
       │    (Svelte 5 Studio)     │                  │      (Axum Gateway)      │
       │          :3000           │                  │          :8080           │
       └──────────────────────────┘                  └──────┬───────────┬───────┘
                                                            │           │
                     ┌──────────────────────────────────────┘           │
                     ▼                                                  ▼
       ┌──────────────────────────┐                  ┌──────────────────────────┐
       │      ferrous-agent       │                  │      ferrous-cqrses      │
       │     (rig-core AI)        │                  │     (Command Handler)    │
       │          :8082           │                  │          :8081           │
       └─────────────┬────────────┘                  └──────────┬───────────────┘
                     │                                          │
       ┌─────────────┴────────────┐                  ┌──────────┴───────────────┐
       ▼                          ▼                  ▼                          ▼
┌─────────────┐            ┌─────────────┐    ┌─────────────┐            ┌─────────────┐
│ ferrous-iam │            │ ferrous-    │    │ ferrous-    │            │ ferrous-    │
│  (Kanidm)   │            │ vectordb    │    │ streaming   │            │ lakehouse   │
│ :8443/:3636 │            │  (Qdrant)   │    │(Iggy+Arroyo)│            │(Delta Lake) │
│             │            │ :6333/:6334 │    │ :8090/:5115 │            │ /data/delta │
└─────────────┘            └─────────────┘    └─────────────┘            └─────────────┘
```

---

## 🏛️ 12-Submodule Topology

| # | Submodule | Container Service | Primary Ports | Healthcheck Probe |
| :-: | :--- | :--- | :---: | :--- |
| **1** | `ferrous-iam` | `kanidmd` | `8443`, `3636` | `kanidmd healthcheck` |
| **2** | `ferrous-lakehouse` | `lakehouse-init` | Volume Mount | `init-tables.sh --check` |
| **3** | `ferrous-streaming` | `iggy-server`, `arroyo` | `8090`, `5115` | `iggy-ping`, `arroyo-health` |
| **4** | `stream-replay-rs` | `traffic-replayer` | CLI / Daemon | Replayer heartbeat |
| **5** | `ferrous-query-engine` | `datafusion-server` | `50051`, `8085` | `SELECT 1` heartbeat probe |
| **6** | `ferrous-vectordb` | `qdrant` | `6333`, `6334` | `GET /readyz` HTTP 200 |
| **7** | `ferrous-cqrses` | `command-handler` | `8081` | Socket / Partition liveness |
| **8** | `ferrous-api` | `axum-gateway` | `8080` | `GET /health/ready` HTTP 200 |
| **9** | `ferrous-agent` | `agent-service` | `8082` | `GET /health` HTTP 200 |
| **10** | `ferrous-frontend` | `web-studio` | `3000` | `GET /healthz` HTTP 200 |
| **11** | `ferrous-reverseproxy` | `zentinel` | `80`, `443` | TLS port 443 handshake |
| **12** | `ferrous-docker` | `jaeger` (Ephemeral) | `16686`, `4317` | `GET /` HTTP 200 |

---

## 🛠️ Developer Quickstart

```bash
# Launch full 12-submodule platform locally
docker compose up -d

# View real-time status and health checks
docker compose ps

# Inspect distributed traces in Jaeger UI
open http://localhost:16686

# Run automated end-to-end platform smoke test
./scripts/smoke-test.sh
```

---

## 🚀 Local Demo Stack: Provision Services & Run `stream-replay-rs`

[`docker-compose.yml`](docker-compose.yml) in this repo runs four containers from one Compose file: `ferrous-iam` (Kanidm), `ferrous-lakehouse` (Delta Lake initializer), `iggy-broker` (Apache Iggy) and `arroyo` (stream engine). Run every command below from the **workspace root** (the `ferrouscx-bmad` checkout that contains this repo as the `ferrous-docker/` submodule), and follow these steps to bring the stack up and drive them with the `stream-replay-rs` CLI for a local demo or manual testing.

### Prerequisites

| Tool | Version | Used for |
| :--- | :--- | :--- |
| Docker + Compose | `v2.29+` | Building images and running the stack |
| Rust | `1.80+` | Building `stream-replay-rs` |
| `openssl` | any | Generating Kanidm dev TLS certificates (bundled with Git for Windows) |
| Python | `3.9+` | `ferrous-streaming` provisioning scripts |

Make sure the submodules are checked out first (`git submodule update --init --recursive` in the workspace root).

> [!IMPORTANT]
> **Where and in which shell to run these commands**
> - Start every step from the **workspace root** (`ferrouscx-bmad/`), not from inside `ferrous-docker/`. Paths such as `ferrous-docker/.env` are relative to the root. Where a step says `cd`, return to the root afterwards.
> - **macOS / Linux:** use the `bash` blocks as written.
> - **Windows:** use **Git Bash** for the `bash` blocks, or **PowerShell** with the `powershell` blocks where given. `cmd.exe` can only run the `.bat` launchers (`init-env.bat`, `preflight.bat`); it cannot run the Bash or PowerShell blocks, so open Git Bash or PowerShell for those.

> [!IMPORTANT]
> Stop any other FerrousCX stack first (`docker compose down` in `ferrous-streaming`, or `stream-replay-rs/docker-compose.dev.yml`). They reuse ports `3000`, `8090` and `9092` and the container names `ferrous-iggy-broker` / `ferrous-arroyo-streaming`.

### 1. Build the Local Images

The Compose file references images that are **not published to a registry**, so build all four locally:

```bash
docker build -t ferrous-iam:latest ferrous-iam
docker build -t ferrous-lakehouse:latest ferrous-lakehouse

# Iggy and Arroyo (the Dockerfiles read config/ and pipelines/ from their build context)
(cd ferrous-streaming && ./scripts/build-and-scan.sh --skip-scan)
```

PowerShell: `Push-Location ferrous-streaming; .\scripts\build-and-scan.ps1 -SkipScan; Pop-Location`.

`build-and-scan` builds `ferrous-streaming-iggy:latest` and `ferrous-streaming-arroyo:latest`. Drop `--skip-scan` to also run the Trivy vulnerability gate.

### 2. Generate Dev TLS Certificates

Kanidm will not boot without TLS material. Generate self-signed certs and place them where the Compose file mounts them (`ferrous-docker/config/certs/chain.pem` and `key.pem`):

```bash
./ferrous-iam/scripts/generate-dev-certs.sh      # Windows: powershell -ExecutionPolicy Bypass -File ferrous-iam/scripts/generate-dev-certs.ps1
mkdir -p ferrous-docker/config/certs
cp ferrous-iam/config/certs/chain.pem ferrous-iam/config/certs/key.pem ferrous-docker/config/certs/
```

PowerShell equivalent:

```powershell
powershell -ExecutionPolicy Bypass -File ferrous-iam\scripts\generate-dev-certs.ps1
New-Item -ItemType Directory -Force ferrous-docker\config\certs | Out-Null
Copy-Item ferrous-iam\config\certs\chain.pem, ferrous-iam\config\certs\key.pem ferrous-docker\config\certs\
```

`ferrous-docker/config/` already holds `server.toml` (Kanidm), `iggy.toml` and `arroyo.toml`.

### 3. Create `ferrous-docker/.env`

Iggy signs its admin JWTs with a shared secret, and the lakehouse service needs a token. Both default to **empty** when unset, and the provisioning scripts in step 5 must use the same Iggy secret as the broker. One script creates the file with a random secret; it works from any directory and any shell, including `cmd.exe`:

```bash
# macOS / Linux / Git Bash
./ferrous-docker/scripts/init-env.sh
```

```bat
:: Windows: cmd.exe, PowerShell or Git Bash (launches init-env.ps1, so keep both files together)
ferrous-docker\scripts\init-env.bat
```

It writes `ferrous-docker/.env` with `IGGY_HTTP_JWT_ENCODING_SECRET` and `IGGY_HTTP_JWT_DECODING_SECRET` (identical, as HS256 requires), `STREAMING_SERVICE_ACCOUNT=svc_ferrous_streaming` and `LAKEHOUSE_SERVICE_TOKEN=lakehouse_dev_token_2026`. It will not overwrite an existing `.env` unless you pass `--force` (`-Force` for the `.bat`). `.env` holds secrets, so keep it out of Git. `LAKEHOUSE_SERVICE_TOKEN` can be any local string of 8+ characters, or a token from `ferrous-iam/scripts/get-service-token.sh` (see step 5).

### 4. Start the Stack

```bash
cd ferrous-docker
docker compose up -d
docker compose ps        # wait until ferrous-iam, iggy-broker and arroyo report "healthy"
cd ..
```

`arroyo` waits for `iggy-broker` to become healthy. `ferrous-lakehouse` publishes no port and has no health gate, so confirm it initialized with `docker logs ferrous-lakehouse`.

| Service | Container | Endpoint |
| :--- | :--- | :--- |
| Kanidm (IAM) | `ferrous-iam` | `https://localhost:8443` (self-signed cert), LDAPS `3636` |
| Apache Iggy HTTP | `ferrous-iggy-broker` | `http://localhost:3000/ping` |
| Kafka gateway (**not served**: Iggy 0.9.0 has none) | `ferrous-iggy-broker` | `localhost:9092` is published, but nothing listens behind it |
| Apache Iggy native TCP | `ferrous-iggy-broker` | `localhost:8090` |
| Arroyo console | `ferrous-arroyo-streaming` | `http://localhost:5115` |
| Delta Lake | `ferrous-lakehouse` | volume `ferrous_lakehouse_data` (`/data/delta`) |

### 5. Provision the Services

**Iggy topics and Arroyo pipelines.** Load the same Iggy secret that is in `ferrous-docker/.env` into your shell, then run the `ferrous-streaming` provisioners. They create stream `orders` with topics `intake` and `consolidated` (4 partitions each) and the consumer groups, then deploy the two Arroyo pipelines.

```bash
# macOS / Linux / Git Bash: load the secret from .env, then provision
export IGGY_HTTP_JWT_ENCODING_SECRET=$(grep '^IGGY_HTTP_JWT_ENCODING_SECRET=' ferrous-docker/.env | cut -d= -f2-)

cd ferrous-streaming
./scripts/provision-topics.sh
./scripts/provision-consumer-groups.sh
./scripts/provision-arroyo.sh
./scripts/run-migrations.sh        # deploys raw_event_archiver and state_snapshot_consolidator
cd ..
```

```powershell
# Windows PowerShell: load the secret from .env, then run the .ps1 twin of each script
$env:IGGY_HTTP_JWT_ENCODING_SECRET = ((Get-Content ferrous-docker\.env | Where-Object { $_ -like 'IGGY_HTTP_JWT_ENCODING_SECRET=*' }) -split '=', 2)[1]

Push-Location ferrous-streaming
.\scripts\provision-topics.ps1
.\scripts\provision-consumer-groups.ps1
.\scripts\provision-arroyo.ps1
.\scripts\run-migrations.ps1
Pop-Location
```

The scripts are idempotent, so re-running is safe. The exported variable only lasts for the current shell window: load it again in a new window.

> [!WARNING]
> **Known state of step 5 against the Iggy 0.9.0 image this stack builds** (`Dockerfile.iggy` pulls `apache/iggy:latest`, which is not pinned):
> - `provision-topics` and `provision-consumer-groups` work only with the `ferrous-streaming` fixes that change the admin token's user id from `1` to `0` (Iggy 0.9.0 treats `1` as an unknown user and answers `403`), add the now-required `compression_algorithm` / `max_topic_size` topic fields, base64-encode the partition id, and address streams and topics by name. Without those fixes they stop at `403 unauthorized`.
> - `provision-arroyo` and `run-migrations` do **not** work. Arroyo reads from Iggy through a Kafka connector on port `9092`, and Iggy 0.9.0 does not serve one (see the endpoint table above and the preflight check).
> - The `ferrous-streaming` unit tests use mock brokers written for the older API, so they report additional failures after those fixes.

**Kanidm service accounts (optional for the demo).** The replayer runs without them (see the note in step 6). To create the six M2M accounts, including `svc_stream_replay`:

```bash
./ferrous-iam/scripts/provision-m2m.sh                              # requires jq; run once ferrous-iam is healthy
./ferrous-iam/scripts/get-service-token.sh svc_ferrous_lakehouse    # optional: generates a local dev token file (not issued by Kanidm)
```

See [`ferrous-iam`](https://github.com/reidlai/ferrous-iam) for the admin password helper (`get-admin-password.sh`) and the human personas (`provision-human-rbac.sh`).

### 6. Run `stream-replay-rs`

Run the read-only pre-flight check first. It verifies container health, `.env` secrets, the Kanidm TLS endpoint, Iggy login and the `orders/intake` topic, the Kafka gateway, Arroyo, the lakehouse container and the replayer binary, and exits non-zero on any failure:

```bash
# macOS / Linux / Git Bash
./ferrous-docker/scripts/preflight.sh
SVC_STREAM_REPLAY_TOKEN=<kanidm token> ./ferrous-docker/scripts/preflight.sh   # also validates the svc_stream_replay token
```

```bat
:: Windows (cmd.exe); preflight.bat launches preflight.ps1, so keep both files together
ferrous-docker\scripts\preflight.bat
set SVC_STREAM_REPLAY_TOKEN=<kanidm token>
ferrous-docker\scripts\preflight.bat
```

The `.sh` script needs `bash`, `curl`, `docker` and (for JSON checks) `python3`; the Windows launcher needs Windows PowerShell 5.1+, `curl.exe` (built into Windows 10 1803+) and `docker`.

Then build and run the replayer:

```bash
cd stream-replay-rs
cargo build --release --features kafka

# Smoke test with no broker involved
./target/release/stream-replay-rs \
  --input-file-path data/olist_orders.parquet \
  --sink stdout --speed 10x --progress

# Point at the Iggy Kafka gateway from the stack
./target/release/stream-replay-rs \
  --input-file-path data/olist_orders.parquet \
  --sink kafka --sink-url localhost:9092 \
  --topic orders/intake --speed 10x --progress
```

On Windows use `.\target\release\stream-replay-rs.exe`. Micro-burst and looping variants are in the [`stream-replay-rs` README](https://github.com/reidlai/stream-replay-rs). To use a Kanidm M2M account instead of the offline fallback, add `--auth-url https://localhost:8443 --svc-account svc_stream_replay --svc-secret <secret>`.

> [!WARNING]
> **No Kafka endpoint exists, and the `kafka` sink is a stub.** Port `9092` is published by Docker but nothing in the Iggy 0.9.0 container listens on it (`preflight` reports this as a failure).
>
> **Current limitation of the `kafka` sink.** In the `stream-replay-rs` revision pinned by this workspace, the Kafka/Iggy sink only opens a TCP connection to the gateway and counts offsets locally; it does not write records to the wire. A successful run logs `Successfully established TCP connection to Kafka gateway`, which proves the stack is reachable, but **no events will appear in Iggy, Arroyo or Delta Lake**. Likewise, the M2M step falls back to a mock token when no Kanidm secret is supplied. See `stream-replay-rs/src/sink/kafka.rs` before treating the demo as end-to-end.

### 7. Check the Stack

```bash
curl http://localhost:3000/ping            # Iggy: "pong"   (PowerShell: use curl.exe, since curl is an alias for Invoke-WebRequest)
curl http://localhost:5115/api/v1/ping     # Arroyo health
docker logs ferrous-lakehouse              # Delta table initialization
# Arroyo pipelines UI: http://localhost:5115/pipelines
```

> [!NOTE]
> Arroyo writes Delta output to the host bind mount `ferrous-docker/data/delta`, while `ferrous-lakehouse` uses the named volume `ferrous_lakehouse_data`. These are **different storage locations**, so tables created by the lakehouse initializer are not the files Arroyo writes to. Inspect Arroyo output under `ferrous-docker/data/delta`.

### 8. Stop and Reset

```bash
cd ferrous-docker
docker compose down            # stop, keep data
docker compose down -v         # also delete the Kanidm and lakehouse named volumes
# Iggy and Arroyo state lives in ferrous-docker/data/{iggy,arroyo,delta}; delete those folders for a clean slate
```

---

## 🛡️ DevSecOps & Smoke Testing

- Multi-Submodule Hygiene: Pre-commit git hooks run `hadolint` and `trivy` on all 12 Dockerfiles.
- Zero-Privilege Runtime: All containers run as non-root users with read-only root filesystems where applicable.
