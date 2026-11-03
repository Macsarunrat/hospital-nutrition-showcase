# System Architecture & Infrastructure Specification

This document details the architecture, component boundaries, data lifecycles, and operational characteristics of the Hospital Nutrition System.

---

## 1. Component Topology & Network Boundaries

```mermaid
graph TB
    subgraph Perimeter ["Perimeter Tier (External Network)"]
        User(["Hospital Clinician<br/>(Desktop / Tablet)"])
        Gateway["Institutional Nginx Reverse Proxy<br/>(TLS Termination & Host Firewall)"]
    end

    subgraph Edge ["Edge Ingress Tier (Host)"]
        Traefik["Traefik v2.11 Proxy<br/>(:80)"]
        SocketProxy["tecnativa/docker-socket-proxy<br/>(Read-Only Socket Daemon)"]
    end

    subgraph InternalApp ["Application Tier (Internal Bridge: traefik-internal)"]
        Frontend["Frontend Web Server<br/>(Nginx serving static SPA)"]
        DjangoCore["Django REST API<br/>(WSGI / WhiteNoise)"]
        FastAPIService["FastAPI AI Service<br/>(Async Image Inference)"]
    end

    subgraph WorkerTier ["Asynchronous Compute Tier"]
        Redis[("Redis Broker & Cache<br/>(LRU Cap: 256MB)")]
        CeleryWorker["Celery Worker<br/>(Concurrency: 2, Max Tasks: 1000)"]
        Flower["Flower Dashboard<br/>(127.0.0.1:5555)"]
    end

    subgraph Persistence ["Persistence Tier"]
        Postgres[("PostgreSQL 16 + pgvector")]
        MediaStorage[("Media Volume Storage<br/>(Patient Meal Photos)")]
    end

    User -->|HTTPS| Gateway
    Gateway -->|Internal LAN /hospital-nutrition/| Traefik
    SocketProxy -.->|TCP 2375 (Read-only)| Traefik

    Traefik -->|PathPrefix /| Frontend
    Traefik -->|PathPrefix /api/| DjangoCore

    DjangoCore -->|DB Query| Postgres
    DjangoCore -->|Store Raw Uploads| MediaStorage
    DjangoCore -->|Enqueue Job| Redis

    Redis -->|Consume Task| CeleryWorker
    CeleryWorker -->|Inference Call| FastAPIService
    CeleryWorker -->|Store Result & Macros| Postgres
    CeleryWorker -->|Write Cropped Visuals| MediaStorage

    Flower -.->|Telemetry| Redis
```

---

## 2. Service Responsibilities

### 2.1 Core API Layer (`backend-django`)
- **Framework:** Django 5.x / Django REST Framework
- **Primary Roles:**
  - Clinical authentication, role-based access control (Doctor, Nurse, Admin), and HttpOnly JWT issuance.
  - CRUD operations on patient admissions, meal schedules, and nutritional targets.
  - Transaction management and ACID compliance over PostgreSQL.
  - Coordinating Celery task dispatches when new meal images are uploaded.
- **Static Assets:** Handled internally via WhiteNoise (`CompressedManifestStaticFilesStorage`) with build-time `collectstatic`.

### 2.2 Async Worker Layer (`celery` + `redis`)
- **Message Broker:** Redis Alpine (`--maxmemory 256mb --maxmemory-policy allkeys-lru --appendonly yes`).
- **Worker Configuration:** `celery -A nutritionWebsite worker -l INFO --concurrency=2 --max-tasks-per-child=1000`.
- **Why Celery was introduced:**
  Meal photo analysis involves YOLO food detection, bounding box crops, and regression-based calorie estimation. Processing synchronously in the Django request thread would result in 5–15 second HTTP timeouts for clinical users on hospital tablets. Celery converts this into a fire-and-poll asynchronous pattern:
  1. Clinician uploads photo -> API responds with HTTP 202 Accepted + `task_id` in <150ms.
  2. Celery executes heavy computation in the background.
  3. Client polls or awaits status update.

### 2.3 AI Inference Microservice (`backend-fastapi`)
- **Runtime:** FastAPI / Uvicorn (ASGI).
- **Primary Role:** Dedicated to loading machine learning weights, executing Tensor operations, and returning structured JSON inference outputs.
- **Inter-service Security:** Protected by a shared bearer token (`INTERNAL_TOKEN`). It is only reachable across the private `traefik-internal` network; external clients cannot query it directly.

### 2.4 Database & Vector Storage (`nutrition_db`)
- **Engine:** PostgreSQL 16 with `pgvector`.
- **Capabilities:** Standard relational models for clinical records, combined with vector similarity search for nutritional meal indexing and menu suggestions.

---

## 3. Network Isolation & Security Perimeter

1. **Dual Network Topologies:**
   - `default` bridge: Only containers that must receive incoming external HTTP connections (`traefik`, `frontend`, `flower`) join this network.
   - `traefik-internal` bridge: Flagged with `internal: true`. Containers here (`db`, `redis`, `fastapi`, `django`, `celery`) communicate with each other, but Docker prevents the daemon from routing traffic from host interfaces directly to them.
2. **Localhost-Only Bindings:**
   - Management dashboards like Flower (Celery monitor) and Traefik's internal metrics dashboard are bound to `127.0.0.1`. They are never mapped to `0.0.0.0`, eliminating inadvertent public exposure of operational telemetry.
3. **Healthcheck-Driven Boot Sequencing:**
   - Docker Compose does not natively wait for databases to finish initialization before launching application containers.
   - We implement native healthchecks on PostgreSQL (`pg_isready`) and Redis (`redis-cli ping`), combined with `depends_on: { condition: service_healthy }` on downstream services. This guarantees zero race-condition crashes during cold stack reboots.
