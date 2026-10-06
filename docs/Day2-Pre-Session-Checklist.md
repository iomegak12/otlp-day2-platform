# Day 2 Pre-Session Checklist (Instructor Only)

Do this on your demo laptop **the day before**, and the first build on participants' machines **before the session starts**. The first build downloads several gigabytes.

## 1. Machine

- [ ] Docker Desktop (or Docker Engine) with Compose v2: `docker compose version`
- [ ] **10–12 GB of memory for Docker** (Docker Desktop → Settings → Resources). The stack runs about 25 containers.
- [ ] About 15 GB of free disk space
- [ ] Free local ports: 3000, 3100, 3200, 4317, 4318, 8000, 8025, 8080, 9090, 9091, 9093
- [ ] A bash shell for the scenario scripts (macOS/Linux terminal, or Git Bash / WSL on Windows)

## 2. Network access during the first build

- [ ] Docker Hub (all images except telemetrygen)
- [ ] GitHub Container Registry `ghcr.io` (telemetrygen, scenario 2)
- [ ] Maven Central `repo.maven.apache.org` (trade-api build and the Java agent)
- [ ] PyPI `pypi.org` and `files.pythonhosted.org` (Python services)

Behind a corporate proxy, set the proxy in Docker Desktop and pass it to builds: `docker compose build --build-arg HTTPS_PROXY=http://proxy:port`. This covers the Python builds. Maven (trade-api) ignores it: add `ENV MAVEN_OPTS="-Dhttps.proxyHost=PROXY -Dhttps.proxyPort=PORT"` to the build stage of `services/trade-api/Dockerfile`, or build trade-api once on a network with direct access.

## 3. Build and pull (10–20 minutes the first time)

```bash
cd tradenova-day2-platform
docker compose --profile tools --profile spike pull --ignore-buildable
docker compose --profile tools build
docker compose up -d
```

- [ ] `docker compose ps`: every service is `running`. `kafka-init` and `fleet-init` show `exited (0)`; they run once.

## 4. Smoke test (after about 5 minutes of running)

| Check | How | Expected |
| --- | --- | --- |
| Services answer | `curl localhost:8080/health` and `curl localhost:8000/health` | `ok` |
| Load is flowing | `docker compose logs --tail 3 loadgen` | `... requests sent` |
| Traces | Grafana → Explore → Tempo → Search, service `trade-api` | traces listed |
| Metrics | Prometheus: `http_server_request_duration_seconds_count` | series for `tradenova/trade-api` and `tradenova/portfolio-service` |
| Logs | Grafana → Explore → Loki: `{service_name="trade-api"}` | log lines (with PII: that is the starting state) |
| Gateway and Kafka | `docker compose logs otel-gateway` | no errors; topic `tradenova-logs` listed in `docker compose logs kafka-init` |
| Fake AWS | `docker compose run --rm fleet status` | 3 running market-data instances |
| Dashboards | Grafana → Dashboards → TradeNova | 4 dashboards |
| Inbox | http://localhost:8025 | Mailpit opens (empty) |
| Batch job | `docker compose logs --tail 5 eod-reconciliation` | runs every 90 s, "Metrics push disabled" |

## 5. Rehearse the two riskiest demos

- [ ] **Scenario 2** (OOM kill). Run `scenarios/02-market-open-spike/trigger-spike.sh` once in the starting state and confirm the restart count goes up. If your laptop never reaches the limit, use the fallback in the runbook (the "queue is full" log lines).
- [ ] **Scenario 5** (Kafka buffer). Apply scenario 5, run a 60-second outage (`scenarios/05-no-lost-logs/loki-outage.sh 60`) and check that the lag rises and returns to 0.

Then put everything back:

```bash
scripts/reset-to-start.sh
```

## 6. Ten minutes before the class

- [ ] `docker compose up -d` (if the laptop was restarted)
- [ ] `scripts/reset-to-start.sh`
- [ ] Open the browser tabs listed at the top of the runbook
- [ ] Open the speaker script and the runbook side by side
- [ ] Clear the Mailpit inbox (trash icon) so scenario 8 starts empty
