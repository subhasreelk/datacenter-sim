# datacenter-sim
Scalable multi-user login load testing framework — Locust + FastAPI + Prometheus + Grafana, built to simulate millions of concurrent users.
# Data Centre Login Load Simulation Framework

> A scalable, open-source framework for simulating real-world multi-user login
> traffic against a server — built to test how authentication systems behave
> under realistic ramp-up, peak, and sustained concurrency conditions, from
> hundreds to millions of users.

![Python](https://img.shields.io/badge/python-3.11-blue)
![Locust](https://img.shields.io/badge/load%20testing-locust-brightgreen)
![FastAPI](https://img.shields.io/badge/server-fastapi-009688)
![Docker](https://img.shields.io/badge/infra-docker-2496ED)
![Prometheus](https://img.shields.io/badge/monitoring-prometheus-E6522C)
![Grafana](https://img.shields.io/badge/dashboards-grafana-F46800)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## What this is

Most server load tests send flat, constant traffic — but real production traffic
ramps up, spikes, sustains, and cools down, often through multiple interfaces at
once, with different user types competing for the same server resources. This
framework reproduces that realism around a **login/authentication system**, one
of the most concurrency-sensitive parts of any application.

It simulates **three interface types — web, mobile, and API — each with light
and heavy user variants**, generating realistic session lifecycles (login →
profile read/update → logout) and sustained machine traffic side by side. This
creates genuine **cross-interface contention**: a heavy API client hammering
`/api/data` while web and mobile users are simultaneously logging in and
refreshing sessions on the same server.

Two time-based traffic patterns are included — a steady **normal day** baseline
and a sudden **flash event** spike — while monitoring the target server live and
automatically identifying the exact point where it starts to break down.

---

## Architecture

```
                    ┌──────────────────────┐
                    │  workload_profiles/    │
                    │  normal_day.yaml       │  ← steady baseline pattern
                    │  flash_event.yaml      │  ← sudden spike pattern
                    └────────┬───────────────┘
                             │
                    ┌────────▼─────────┐
                    │   Locust          │  ← WebUser, MobileUser, APIUser
                    │ (master + workers)│     each with light/heavy variants
                    └────────┬─────────┘     scales horizontally
                             │ HTTP requests
                    ┌────────▼──────────────────────┐
                    │  FastAPI server                │
                    │  /login /logout /me             │
                    │  /profile/update /api/data      │
                    │  /health /metrics                │
                    │   (test target)                  │
                    └────────┬──────────────────────┘
                             │ metrics
          ┌──────────────────┼──────────────────┐
          ▼                                       ▼
   ┌─────────────┐                       ┌──────────────┐
   │ Prometheus  │──────────────────────▶│   Grafana    │
   │ + Node Exp. │     live dashboards   │  dashboards   │
   └─────────────┘                       └──────────────┘
          │
          ▼
   ┌─────────────────────┐
   │ analysis/analyse.py  │ ← Pandas: bottleneck detection,
   │ → PDF report          │   SLA verdict, charts
   └─────────────────────┘
```

---

## Tech stack

| Layer | Tools |
|---|---|
| Language & config | Python 3.11, YAML, Git |
| Test target server | FastAPI, Uvicorn |
| Load generation | Locust (distributed master/worker) |
| Infrastructure | Docker, Docker Compose |
| Monitoring | Prometheus, Node Exporter, Grafana |
| Analysis & reporting | Pandas, Matplotlib, Seaborn, WeasyPrint |

All tools are 100% open source — zero licensing cost.

---

## Server endpoints

| Method | Endpoint | Used by | Purpose |
|---|---|---|---|
| POST | `/login` | All users | Authenticate, issue session token |
| POST | `/logout` | Web users | End session, invalidate token |
| GET | `/me` | Web + Mobile | Get current user profile (light read) |
| POST | `/profile/update` | Mobile (heavy), Web (heavy) | Update profile — write-heavy, DB contention |
| GET | `/api/data` | API users | Sustained machine traffic — resource contention |
| GET | `/health` | Monitoring | Liveness check |
| GET | `/metrics` | Prometheus | Exposes request count, latency histograms |

## Simulated user types

Three interface types are simulated, each split into weighted **light** and
**heavy** variants to model realistic variability in user behaviour:

| User type | Weight | Light variant | Heavy variant |
|---|---|---|---|
| **WebUser** | 50% | 80% — login → `/me` → logout, single pass | 20% — login → loop(`/me` + `/profile/update`) → logout |
| **MobileUser** | 35% | 60% — login → `/me` once | 40% — login → poll `/me` repeatedly (background sync) |
| **APIUser** | 15% | 30% — login once → occasional `/api/data` | 70% — login once → sustained `/api/data` calls |

The heavy `APIUser` variant is the primary source of **cross-interface
contention** — it competes for the same server resources (CPU, session store)
as web and mobile traffic on `/login` and `/me`.

---

## Folder structure

```
datacenter-sim/
├── app/                     # FastAPI server (test target)
│   ├── main.py              # /login /logout /me /profile/update /api/data /health /metrics
│   ├── Dockerfile
│   └── requirements.txt
├── simulation/               # Locust load generation
│   ├── locustfile.py         # WebUser, MobileUser, APIUser (light + heavy variants)
│   └── loadshapes.py         # ramp-up / peak / cooldown curves
├── infra/                     # Infrastructure
│   ├── docker-compose.yml
│   └── prometheus.yml
├── monitoring/                # Grafana dashboards
│   └── dashboards/*.json
├── analysis/                  # Analysis & reporting
│   ├── analyse.py
│   └── reports/
├── workload_profiles/         # Test scenario configs
│   ├── normal_day.yaml       # steady baseline (day/night cycle pattern)
│   └── flash_event.yaml      # sudden spike (event-driven surge pattern)
├── docs/                       # Whitepaper, diagrams, notes
├── .gitignore
└── README.md
```

---

## Quick start

### 1. Clone and install

```bash
git clone https://github.com/yourname/datacenter-sim.git
cd datacenter-sim
pip install -r app/requirements.txt
```

### 2. Start the full stack

```bash
cd infra
docker-compose up --build
```

This starts: the FastAPI login server, Locust master + workers, Prometheus,
Node Exporter, and Grafana — all on one Docker network.

### 3. Open the dashboards

| Service | URL |
|---|---|
| FastAPI server | http://localhost:8000 |
| Locust UI | http://localhost:8089 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 |

### 4. Run a test

Run the steady baseline first, then the spike scenario, so the analysis
phase can compare them:

```bash
python simulation/run_test.py --profile workload_profiles/normal_day.yaml
python simulation/run_test.py --profile workload_profiles/flash_event.yaml
```

### 5. Scale up workers (for higher simulated user counts)

```bash
docker-compose up --scale locust-worker=4
```

### 6. Generate the report

```bash
python analysis/analyse.py --run results/latest.csv
```

Outputs a PDF report to `analysis/reports/` with latency/throughput charts,
bottleneck detection, and an SLA pass/fail verdict.

---

## Scalability approach

"Scalable to millions of users" is achieved architecturally, not by literally
running a million-user test on a laptop:

- **Config-driven scale** — `target_users` in the YAML profile controls the
  entire test; changing one number changes scale.
- **Locust distributed mode** — one master coordinates many worker nodes, each
  generating load independently.
- **Horizontal scaling** — `docker-compose up --scale locust-worker=N` adds
  capacity without code changes.
- **Bottleneck identification** — analysis identifies the exact ceiling (CPU,
  DB connections, thread pool, etc.) that limits scale, and what to fix to go higher.

---

## SLA thresholds (default)

| Metric | Threshold |
|---|---|
| p95 login latency | < 500ms |
| Error rate | < 1% |
| CPU utilisation | < 80% |
| Memory utilisation | < 90% |

Configurable per-scenario in `workload_profiles/normal_day.yaml` and
`workload_profiles/flash_event.yaml`.

---

## Team

| Role | Responsibility |
|---|---|
| App developers (2) | FastAPI login server |
| Simulation developer (1) | Locust load generation, scaling |
| Monitoring & analysis (3) | Docker infra, Grafana dashboards, bottleneck tuning, reporting |

---

## License

MIT — free to use, modify, and distribute.
