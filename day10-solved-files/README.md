# ReconX — Enterprise Trade Reconciliation Platform

ReconX is a full-stack trade reconciliation system built during the Advanced Track of DB26.
It ingests trade events via Kafka, persists them to PostgreSQL, detects reconciliation breaks,
and surfaces real-time alerts and dashboards through a React SPA backed by a Spring Boot API.

---

## Architecture

See [`db/diagrams/architecture.md`](db/diagrams/architecture.md) for full C4 diagrams
(Context → Container → Component levels).

```
Browser (React SPA)
      │  HTTPS :5173
      ▼
  nginx proxy  ──/api/──►  Spring Boot API :8081
                                  │
                   ┌──────────────┼──────────────┐
                   ▼              ▼              ▼
             PostgreSQL 16    Apache Kafka    Prometheus
              (trades, recon,  (trade-events,   (metrics
               audit, envers)   recon-results,   scrape)
                                audit-events,       │
                                dlq)            Grafana
```

---

## Quick Start (Docker Compose)

**Prerequisites:** Docker Desktop 4.x+, at least 4 GB RAM allocated to Docker.

```bash
# 1. Build the backend JAR (Dockerfile picks up target/*.jar)
cd backend && ./mvnw -q clean package -DskipTests && cd ..

# 2. Start the full 7-service stack
docker compose up -d --build

# 3. Watch services become healthy (~30-60s)
watch 'docker compose ps'
```

Once all services show `(healthy)`:

| Service | URL |
|---------|-----|
| Frontend SPA | <http://localhost:5173> |
| Backend Swagger UI | <http://localhost:8081/api/swagger-ui.html> |
| Prometheus | <http://localhost:9090> |
| Grafana | <http://localhost:3000> (admin / admin) |
| Kafdrop (debug profile) | `docker compose --profile debug up -d` → <http://localhost:9000> |

**Tear down (removes volumes):**
```bash
docker compose down -v
```

---

## Running Each Layer

### Backend only (dev mode)
```bash
cd backend
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
# Swagger: http://localhost:8081/api/swagger-ui.html
```

### Frontend only (dev server)
```bash
cd frontend
npm install
npm run dev
# SPA: http://localhost:5173
```

### Tests
```bash
cd backend
./mvnw test                    # unit tests only
./mvnw verify -Pci             # full: tests + JaCoCo 85% gate + Checkstyle
open target/site/jacoco/index.html
```

### k6 Load Test
```bash
# Requires: brew install k6
# Stack must be running first.
k6 run load-tests/trade-creation.js
```

---

## Project Structure

```
reconx/
├── backend/                  Spring Boot 3 / Java 21
│   ├── src/main/java/…       Controllers, Services, Kafka, Security, Observability
│   ├── src/main/resources/   application*.yml, Liquibase changelogs
│   ├── src/test/             Unit + Integration tests (Testcontainers)
│   ├── Dockerfile            Multi-stage: eclipse-temurin:21-jdk → jre-alpine
│   ├── checkstyle.xml        Google-style Checkstyle ruleset
│   └── pom.xml               Dependencies + JaCoCo + Checkstyle plugins
├── frontend/                 React + Vite
│   ├── src/                  Components, hooks, API client
│   ├── Dockerfile            Multi-stage: node:22-alpine → nginx:1.27-alpine
│   └── nginx.conf            SPA fallback + /api/ proxy to backend
├── monitoring/
│   ├── prometheus/           prometheus.yml + alerts.yml
│   └── grafana/provisioning/ Datasource + dashboard JSON
├── load-tests/
│   └── trade-creation.js     k6 script — 200 VUs × 60s
├── scripts/
│   └── smoke-test.sh         Bring-up → health wait → endpoint checks → teardown
├── db/diagrams/              C4 Mermaid architecture diagrams
├── docs/
│   ├── demo-runsheet.md      20-minute demo script
│   └── retro.md              Sprint retrospective
├── docker-compose.yml        Full 7-service stack
└── .github/workflows/ci.yml  GitHub Actions CI pipeline
```

---

## CI / CD

The GitHub Actions workflow (`.github/workflows/ci.yml`) runs on every push to `main` and on version tags:

1. **Liquibase validate** — confirms changelogs are consistent with the schema before any tests run.
2. **Maven verify** (`-Pci`) — runs all tests, enforces the JaCoCo 85% line-coverage gate, and runs Checkstyle.
3. **Docker build & push** — builds backend and frontend images, tags them with the semver or SHA, and pushes to GHCR.

### Release

```bash
# After CI is green on main:
git tag -a v1.0.0 -m "Release 1.0.0 — ReconX Advanced Track"
git push origin v1.0.0
# CI builds ghcr.io/<org>/reconx-backend:1.0.0 and reconx-frontend:1.0.0
```

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Kafka for trade events | Decouples producers from reconciliation consumers; enables replay |
| Dead-letter queue (DLQ) | Poison-pill messages don't block the main pipeline |
| Hibernate Envers | Full audit trail for every trade mutation — no custom trigger SQL |
| Testcontainers for ITs | Tests run against real Postgres + Kafka; no mock divergence |
| JaCoCo 85% gate | Enforces meaningful test coverage in CI; excludes generated mappers/DTOs |
| Checkstyle (Google style) | Keeps code readable across the team; max 120 chars/line |
| Caffeine cache | In-memory cache for instruments and counterparties; evicts after 5 min |
| Micrometer + Prometheus | Zero-overhead metrics via Spring Boot Actuator |

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Backend container exits — `IllegalArgumentException: JWT_SECRET` | Set `JWT_SECRET` in `docker-compose.yml` under `backend.environment` |
| Prometheus panels show "No data" | Confirm `monitoring/prometheus/prometheus.yml` target is `backend:8081`, not `localhost` |
| Grafana dashboards empty | Datasource URL must use `http://prometheus:9090` (service name), not `localhost` |
| JaCoCo below 85% | Add tests for exception paths and empty-result branches |
| k6 reports lots of 401 | Token expires mid-run — call `/auth/login` inside k6 `setup()` (already done in the provided script) |
| `docker compose up` hangs | Run `docker compose logs <service>` for the stuck service |
