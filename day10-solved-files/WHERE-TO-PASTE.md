# Day 10 — Solved Files & Where To Paste

Day 10 is release day: containers, CI, quality gates, load test, docs, and the `v1.0.0` tag.

This folder now contains **complete, drop-in solutions for all 20 tickets (ADV146–ADV165)**.

---

## Quick Start — apply everything at once

```bash
# Run from the project root

# ── Infra / compose ────────────────────────────────────────────────────────
cp day10-solved-files/docker-compose.yml .
cp day10-solved-files/backend/Dockerfile backend/
cp day10-solved-files/frontend/Dockerfile frontend/
cp day10-solved-files/frontend/nginx.conf frontend/
cp -R day10-solved-files/monitoring .

# ── Backend build + quality ────────────────────────────────────────────────
cp day10-solved-files/backend/pom.xml backend/
cp day10-solved-files/backend/checkstyle.xml backend/

# ── CI workflow ────────────────────────────────────────────────────────────
mkdir -p .github/workflows
cp day10-solved-files/.github/workflows/ci.yml .github/workflows/

# ── Load test ─────────────────────────────────────────────────────────────
mkdir -p load-tests
cp day10-solved-files/load-tests/trade-creation.js load-tests/

# ── Smoke test ────────────────────────────────────────────────────────────
mkdir -p scripts
cp day10-solved-files/scripts/smoke-test.sh scripts/
chmod +x scripts/smoke-test.sh

# ── Docs + README ─────────────────────────────────────────────────────────
cp day10-solved-files/README.md .
mkdir -p docs
cp day10-solved-files/docs/demo-deck.md docs/
cp day10-solved-files/docs/demo-runsheet.md docs/
cp day10-solved-files/docs/retro.md docs/

# ── Architecture diagrams ──────────────────────────────────────────────────
mkdir -p db/diagrams
cp day10-solved-files/db/diagrams/architecture.md db/diagrams/
```

---

## What Each File Solves

### Containers + Compose

| File | Ticket | What it does |
|------|--------|--------------|
| `backend/Dockerfile` | ADV146 | Multi-stage build: `eclipse-temurin:21-jdk` compiles the JAR, `eclipse-temurin:21-jre-alpine` runs it |
| `frontend/Dockerfile` | ADV147 | Multi-stage: `node:22-alpine` builds the SPA, `nginx:1.27-alpine` serves it |
| `frontend/nginx.conf` | ADV147 | SPA fallback (`try_files $uri /index.html`) + reverse proxy to `backend:8081/api/` |
| `docker-compose.yml` | ADV148, ADV152 | 7-service stack with healthcheck gates on every service |
| `monitoring/prometheus/prometheus.yml` | ADV149 | Scrapes `backend:8081/api/actuator/prometheus` every 10s |
| `monitoring/prometheus/alerts.yml` | ADV149 | Alert rules: too many breaks, high P95 latency, DLQ growing |
| `monitoring/grafana/provisioning/datasources/prometheus.yml` | ADV150 | Wires Grafana → Prometheus datasource on first start |
| `monitoring/grafana/provisioning/dashboards/reconx.yml` | ADV150 | Grafana dashboard provisioning config |
| `monitoring/grafana/provisioning/dashboards/reconx-overview.json` | ADV150 | ReconX Overview dashboard JSON |

**ADV151** — Already handled: `application.yml` has `spring.liquibase.enabled: true`.
**ADV152** — Already handled: every service in `docker-compose.yml` has a `healthcheck` block.

### CI + Quality Gates

| File | Ticket | What it does |
|------|--------|--------------|
| `.github/workflows/ci.yml` | ADV154, ADV155 | GitHub Actions: Liquibase validate → Maven verify → Docker build & push to GHCR |
| `backend/pom.xml` | ADV156, ADV157 | Adds JaCoCo 85% line-coverage `check` goal + Checkstyle plugin with `ci` profile |
| `backend/checkstyle.xml` | ADV157 | Google-style Checkstyle ruleset (120-char lines, no star imports, braces required) |
| `load-tests/trade-creation.js` | ADV158 | k6 script — 200 VUs × 60s, p(95) < 800ms gate, < 1% error rate |
| `scripts/smoke-test.sh` | ADV153 | Bring up stack → wait for health → auth → POST trade → GET trade → Prometheus check → teardown |

### Observability

**ADV159 — Screenshots** are a manual step:
```bash
# Take 3 screenshots and save in screenshots/
# 1. screenshots/01-stack-healthy.png  — `docker compose ps` all green
# 2. screenshots/02-grafana-under-load.png — Grafana while k6 is running
# 3. screenshots/03-k6-results.png     — k6 terminal summary after run
```

### Docs + Release

| File | Ticket | What it does |
|------|--------|--------------|
| `README.md` | ADV161 | Comprehensive project README: quickstart, architecture, per-layer run instructions, CI/release, troubleshooting |
| `db/diagrams/architecture.md` | ADV160 | Mermaid C4 diagrams at Context, Container, and Component levels — render at mermaid.live |
| `docs/demo-deck.md` | ADV162 | 10-slide demo deck in Markdown — convert to .pptx/.pdf with Marp, Slidev, or Pandoc |
| `docs/demo-runsheet.md` | ADV163 | 20-minute demo runsheet with segment timings, commands, contingency plan |
| `docs/retro.md` | ADV165 | Full retrospective: what worked, what didn't, key learnings |

**ADV164 — Tag v1.0.0** (run after CI is green on `main`):
```bash
git tag -a v1.0.0 -m "Release 1.0.0 — ReconX Advanced Track"
git push origin v1.0.0
# CI builds ghcr.io/<org>/reconx-backend:1.0.0 and reconx-frontend:1.0.0
```

---

## Verify the full stack

```bash
# 1. Build the backend JAR first (the Dockerfile copies target/*.jar)
cd backend && ./mvnw -q clean package -DskipTests && cd ..

# 2. Bring up the stack
docker compose up -d --build

# 3. Check every service is healthy
docker compose ps

# 4. Smoke test (automated)
bash scripts/smoke-test.sh
```

Then open:

| URL | Service |
|-----|---------|
| http://localhost:5173 | Frontend SPA |
| http://localhost:8081/api/swagger-ui.html | Backend Swagger |
| http://localhost:9090 | Prometheus |
| http://localhost:3000 | Grafana (admin/admin) |

```bash
# 5. Quality gates locally
cd backend
./mvnw verify -Pci
open target/site/jacoco/index.html   # must show ≥ 85% Total line coverage

# 6. k6 load test
k6 run load-tests/trade-creation.js

# 7. Tear down
docker compose down -v
```
