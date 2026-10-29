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

## 🛡️ DevSecOps & Smoke Testing

- Multi-Submodule Hygiene: Pre-commit git hooks run `hadolint` and `trivy` on all 12 Dockerfiles.
- Zero-Privilege Runtime: All containers run as non-root users with read-only root filesystems where applicable.
