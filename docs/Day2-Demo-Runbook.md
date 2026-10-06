# Day 2 Demo Runbook (Instructor Only)

Step-by-step instructions for every scenario: what to type, what to click, and what to point out. Use it next to the speaker script. When the script says *[Switch to the runbook: Scenario N, Show the problem]*, come here.

Participants repeat each step on their own machines after you.

---

## Before the session

Follow `docs/Day2-Pre-Session-Checklist.md`. By the time the class starts:

- the stack has been built and started with `docker compose up -d --build`;
- it has been running for at least 10 minutes, so dashboards have data;
- `scripts/reset-to-start.sh` has been run once, so every scenario starts in its "problem" state.

## Browser tabs to keep open

| Tab | URL | Used in |
| --- | --- | --- |
| Grafana | http://localhost:3000 | All scenarios (Explore, dashboards in the TradeNova folder) |
| Prometheus | http://localhost:9090 | 1, 4, 6, 7, 8 (Query, Status → Targets, Alerts) |
| Pushgateway | http://localhost:9091 | 6 |
| Alertmanager | http://localhost:9093 | 8 |
| Mailpit inbox | http://localhost:8025 | 8 |

## Three rules for every scenario

1. Run every command from the `tradenova-day2-platform` folder.
2. To apply a fix, use **`scripts/apply-scenario.sh N`** (Windows: `.\scripts\apply-scenario.ps1 -Scenario N`). It copies the finished files for scenario N and restarts what changed. Before running it, show the room the block being added; it is printed in each section below and in `scenarios/0N-*/after/`.
3. If a demo goes wrong, run the same command again; it is safe to repeat. To start everything over: `scripts/reset-to-start.sh`.

The `.sh` trigger scripts need a bash shell. On Windows, use Git Bash or WSL, or run the raw commands shown next to each script.

## Suggested timing

| Time | Session |
| --- | --- |
| 09:30 – 09:40 | Opening: from pilot to platform |
| 09:40 – 10:10 | Scenario 1: Unlabelled data |
| 10:10 – 10:45 | Scenario 2: Market-open spike |
| 10:45 – 11:00 | Break |
| 11:00 – 11:30 | Scenario 3: PII compliance |
| 11:30 – 12:00 | Scenario 4: Observability cost |
| 12:00 – 12:40 | Scenario 5: No lost logs |
| 12:40 – 13:25 | Lunch |
| 13:25 – 13:55 | Scenario 6: Silent batch failure |
| 13:55 – 14:25 | Scenario 7: Scaling fleet |
| 14:25 – 15:05 | Scenario 8: On-call alerts |
| 15:05 – 15:20 | Break |
| 15:20 – 16:00 | Scenario 9: Slow trades |
| 16:00 – 16:30 | Day 2 close, Q&A, buffer |

---

## Scenario 1: Unlabelled data

**Goal:** add environment, region, cost centre, host and owning team to all telemetry, centrally in the agent.

### Show the problem

1. **Prometheus → Query:** run `target_info`.
   Point out: the only labels are `job`, `instance`, `service_version` and SDK details. There is nothing about environment, region or team.
2. **Grafana → Explore → Loki → Label browser.**
   Point out: only `service_name`, `service_namespace` and `service_instance_id` are available.
3. **Grafana → Explore → Tempo → TraceQL:** run `{ resource.tradenova.team = "trading-core" }`.
   Point out: no results. Nobody can ask "show me everything from the trading-core team".

### Apply the fix

Show the added block in `scenarios/01-unlabelled-data/after/collector/agent/config.yaml`:

```yaml
resource_detection/system:
  detectors: [system]
  system:
    hostname_sources: [os]

resource/platform:
  attributes:
    - { key: deployment.environment.name, value: training, action: upsert }
    - { key: cloud.region, value: ap-south-1, action: upsert }
    - { key: tradenova.cost_center, value: CC-4410, action: insert }

transform/ownership:
  error_mode: ignore
  trace_statements:   # (the same statements for metric_statements and log_statements)
    - 'set(resource.attributes["tradenova.team"], "trading-core") where resource.attributes["service.name"] == "trade-api"'
    - 'set(resource.attributes["tradenova.team"], "wealth-apps") where resource.attributes["service.name"] == "portfolio-service"'
```

Also show the pipelines: `processors: [resource_detection/system, resource/platform, transform/ownership]`.

```bash
scripts/apply-scenario.sh 1
```

### What to point out in the result (wait about 30 seconds)

- [ ] **Prometheus:** `target_info{job="tradenova/trade-api"}` now has `cloud_region="ap-south-1"`, `deployment_environment_name="training"`, `tradenova_team="trading-core"`, `tradenova_cost_center="CC-4410"` and `host_name`.
- [ ] **Loki label browser:** `cloud_region` and `deployment_environment_name` appear. `tradenova_team` does not, because Loki only turns a fixed list of resource attributes into labels; the others become structured metadata. Try `{service_name="trade-api"} | tradenova_team="trading-core"`.
- [ ] **TraceQL:** `{ resource.tradenova.team = "trading-core" }` now returns traces.
- [ ] **Applications:** none of them changed or restarted.

---

## Scenario 2: Market-open spike

**Goal:** show the agent being killed by a traffic spike, then protect it with `memory_limiter`, `batch` and `GOMEMLIMIT`.

### Show the problem

1. Open **Grafana → TradeNova → Platform Overview** and scroll to **Collectors**: *Collector memory*, *Data refused by the agent* and *Exporter queue size*.
2. In a second terminal, watch the agent's memory (limit 320 MiB):
   ```bash
   docker stats tradenova-day2-otel-agent-1
   ```
3. Start the spike. It pauses Tempo and floods the agent with traces for 90 seconds:
   ```bash
   scenarios/02-market-open-spike/trigger-spike.sh
   ```
   Raw commands: `docker compose pause tempo`, then `docker compose --profile spike run --rm telemetrygen`, then `docker compose unpause tempo`.

**Expected:** memory climbs to the 320 MiB limit and the container is killed and restarted. The script ends with `restarts=` one higher than before. In testing, this configuration reached about 1.7 GB with no limit. Every span held in the agent's queue is lost with it.

**If it is not killed** (on a fast machine with little load), show the data loss instead: `docker compose logs otel-agent | grep -i "queue is full"`. The exporter queue filled up and new data was rejected.

### Apply the fix

Show the added blocks in `scenarios/02-market-open-spike/after/collector/agent/config.yaml`:

```yaml
memory_limiter:            # FIRST processor in every pipeline
  check_interval: 1s
  limit_percentage: 75     # of the 320 MiB container limit: hard limit 240 MiB
  spike_limit_percentage: 25   # soft limit 240 - 80 = 160 MiB

batch:                     # LAST processor in every pipeline
  send_batch_size: 2048
  timeout: 2s
```

Show the change in `scenarios/02-market-open-spike/after/.env`: `AGENT_GOMEMLIMIT=220MiB` (the Go runtime memory limit).

```bash
scripts/apply-scenario.sh 2
scenarios/02-market-open-spike/trigger-spike.sh
```

### What to point out in the result

- [ ] `docker stats`: memory rises but levels off below the limit. In testing it peaked at about 256 MB.
- [ ] The script ends with `restarts=0` (the fix recreated the container, so the count started again from zero): the agent survived the spike.
- [ ] *Data refused by the agent* jumps during the spike. The memory_limiter is refusing new data with a retryable error, instead of the whole agent dying.
- [ ] After Tempo resumes, *Spans sent* shows the backlog being delivered.
- [ ] Testing finding to share: memory_limiter alone was not enough. Without `GOMEMLIMIT`, memory still reached about 1.7 GB, because requests are decoded before the limiter can refuse them.

---

## Scenario 3: PII compliance

**Goal:** mask account numbers, e-mail addresses and card numbers in the agent, before they reach any backend.

### Show the problem

1. **Grafana → Explore → Loki:** run
   ```logql
   {service_name=~"trade-api|portfolio-service"} |~ "@|card"
   ```
   Point out the full e-mail addresses, account numbers (`ACC-10001234`) and card numbers (`Funding check passed for card 4532-...`).
2. **Explore → Tempo:** run `{ resource.service.name = "trade-api" && span.tradenova.customer.email != nil }`. Open a trace and show `tradenova.customer.email` and `tradenova.account.number` on the span.

### Apply the fix

Show the added block in `scenarios/03-pii-compliance/after/collector/agent/config.yaml`:

```yaml
transform/pii:
  error_mode: ignore
  log_statements:
    - 'replace_pattern(log.body, "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}", "<email-redacted>")'
    - 'replace_pattern(log.body, "\\b\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}\\b", "<card-redacted>")'
    - 'replace_pattern(log.body, "ACC-\\d{8}", "ACC-********")'
    - 'replace_pattern(log.body, "email=[^&\\s\"]*", "email=<redacted>")'   # URL-encoded e-mails in access logs
  trace_statements:
    - 'delete_key(span.attributes, "tradenova.customer.email")'
    - 'replace_pattern(span.attributes["tradenova.account.number"], "ACC-\\d{8}", "ACC-********")'
    - 'replace_pattern(span.attributes["url.path"], "ACC-\\d{8}", "ACC-********")'
    - 'replace_pattern(span.attributes["url.full"], "ACC-\\d{8}", "ACC-********")'
    - 'replace_pattern(span.attributes["url.full"], "email=[^&]*", "email=<redacted>")'
    - 'replace_pattern(span.attributes["url.query"], "email=[^&]*", "email=<redacted>")'
```

It is added to the `traces` and `logs` pipelines. Point out the `url.*` lines: HTTP instrumentation records URLs automatically, and these carry account numbers and e-mails too.

```bash
scripts/apply-scenario.sh 3
```

### What to point out in the result (wait about 30 seconds; set the time range to *Last 5 minutes*)

- [ ] **Loki:** `Trade accepted for account ACC-******** (customer <email-redacted>)` and `Funding check passed for card <card-redacted>`.
- [ ] **Tempo:** new spans have no `tradenova.customer.email`; `tradenova.account.number`, `url.path` and `url.full` show `ACC-********`; `url.query` shows `email=<redacted>`. Query: `{ span.tradenova.account.number = "ACC-********" }`.
- [ ] **Older data is still unmasked.** Masking protects what comes next; data already stored needs retention or deletion.
- [ ] **Metrics still expose account numbers** in the `account_id` label of `tradenova_trades_placed_total`. Scenario 4 removes it.

---

## Scenario 4: Observability cost

**Goal:** drop health-check traces, DEBUG logs and the per-account metric label.

### Show the problem

1. **Grafana → Platform Overview → Scenario 4 - cost:** *Series of tradenova_trades_placed_total* shows hundreds or thousands of series, one per account.
2. **Explore → Tempo:** run `{ span.url.path = "/health" }`. Two health-check traces arrive every second, one per service.
3. **Explore → Loki:** run `{service_name="trade-api"} | detected_level="debug"`. DEBUG lines are logged for every request.
4. Optional: in **Prometheus**, run `count by (account_id) (tradenova_trades_placed_total)` to show the per-account series.

### Apply the fix

Show the added blocks in `scenarios/04-observability-cost/after/collector/agent/config.yaml`:

```yaml
filter/noise:
  error_mode: ignore
  traces:
    span:
      - 'attributes["url.path"] == "/health"'
  logs:
    log_record:
      - 'severity_number < SEVERITY_NUMBER_INFO'
      - 'IsMatch(body, "GET /health")'

metrics_transform/cost:
  transforms:
    - include: tradenova.trades.placed
      action: update
      operations:
        - action: aggregate_labels
          label_set: [symbol, side]
          aggregation_type: sum
```

`filter/noise` is added to the traces and logs pipelines, and `metrics_transform/cost` to the metrics pipeline.

```bash
scripts/apply-scenario.sh 4
```

### What to point out in the result

- [ ] **Platform Overview, *Noise removed by filter/noise*:** dropped spans and log records per second.
- [ ] **Prometheus:** `count(tradenova_trades_placed_total{account_id=""})` shows about 10 new series (5 symbols × 2 sides). The old per-account series stop updating and disappear from the series panel after about 5 minutes.
- [ ] **Prometheus:** `sum by (symbol) (rate(tradenova_trades_placed_total{account_id=""}[1m]))` gives the same business numbers, far cheaper.
- [ ] **Tempo:** no new `/health` traces. **Loki:** no new DEBUG lines.

---

## Scenario 5: No lost logs

**Goal:** prove that a Loki outage loses logs today, then put Kafka between the agent and Loki and prove that nothing is lost.

### Show the problem

1. **Grafana → Platform Overview → Scenario 5 - logs:** *Log lines per minute, by service*, time range *Last 15 minutes*.
2. Run a 2-minute Loki outage in a second terminal:
   ```bash
   scenarios/05-no-lost-logs/loki-outage.sh 120
   ```
   Raw commands: `docker compose stop loki`, wait 2 minutes, `docker compose start loki`.
3. During the outage, show the agent giving up:
   ```bash
   docker compose logs --since 2m otel-agent | grep -i -E "dropping|failed"
   ```
4. About a minute after Loki is back, refresh the panel. **There is a permanent gap** of about 90 seconds: the outage minus the agent's 30-second retry window.

### Apply the fix

Show the change in `scenarios/05-no-lost-logs/after/collector/agent/config.yaml`. The logs exporter changes from `otlp_http/loki` to:

```yaml
kafka:
  brokers: [kafka:9092]
  logs:
    topic: tradenova-logs
    encoding: otlp_proto
```

Then walk through `collector/gateway/config.yaml`, which is unchanged and running all day:
- the `kafka` receiver uses `group_id: otel-gateway`, `message_marking.after: true`;
- the `otlp_http/loki` exporter has `sending_queue.enabled: false` and `retry_on_failure.max_elapsed_time: 0`;
- there is no batch processor.

```bash
scripts/apply-scenario.sh 5
```

### What to point out in the result

1. Start another outage: `scenarios/05-no-lost-logs/loki-outage.sh 120`.
2. During the outage, run this twice, a minute apart:
   ```bash
   scenarios/05-no-lost-logs/kafka-lag.sh
   ```
   - [ ] The **LAG** column grows: records are waiting safely in Kafka.
3. `docker compose logs --since 1m otel-gateway`:
   - [ ] the gateway is retrying, not dropping.
4. After Loki is back:
   - [ ] **LAG** falls to 0 within seconds;
   - [ ] the *Log lines per minute* panel fills the outage period;
   - [ ] the *Log records sent* panel shows the gateway's burst as it catches up.

---

## Scenario 6: Silent batch failure

**Goal:** make the short-lived end-of-day job visible through the Pushgateway.

### Show the problem

1. ```bash
   docker compose logs --tail 20 eod-reconciliation
   ```
   Point out the runs every 90 seconds, some `FAILED`, and `Metrics push disabled ... nobody will know how this run went`.
2. **Prometheus:** search for `eod_`. There is nothing.
3. **Pushgateway UI** (http://localhost:9091): empty.
4. Ask the room: *"Why not just scrape the job?"* It lives for 4 to 12 seconds, then the process exits.

### Apply the fix

Show the two changes:
- `scenarios/06-silent-batch-failure/after/.env`: `EOD_PUSH_ENABLED=true`
- `scenarios/06-silent-batch-failure/after/infra/prometheus/prometheus.yml`:
  ```yaml
  - job_name: pushgateway
    honor_labels: true
    static_configs:
      - targets: ["pushgateway:9091"]
  ```

Also show `push_result()` in `services/eod-reconciliation/job.py`. It uses `pushadd_to_gateway` and sends `eod_job_last_success_timestamp_seconds` only when a run succeeds.

```bash
scripts/apply-scenario.sh 6
```

The job restarts and runs immediately.

### What to point out in the result

- [ ] **Pushgateway UI:** group `job="eod_reconciliation"` with its metrics and `push_time_seconds`.
- [ ] **Prometheus:** `eod_job_success` and `time() - eod_job_last_success_timestamp_seconds`. If the first run failed, there is no success timestamp yet; wait for a successful run.
- [ ] **Grafana → TradeNova → Batch Jobs:** the dashboard is now alive.
- [ ] **Why pushadd matters.** Make every run fail:
  ```bash
  scenarios/06-silent-batch-failure/failure-mode.sh always
  ```
  - *Last run* turns FAILED.
  - *Time since last successful reconciliation* keeps growing, because the old success timestamp is kept.
  - Set it back afterwards: `scenarios/06-silent-batch-failure/failure-mode.sh random`. Scenario 8 makes it fail again.
- [ ] **`honor_labels`:** in Prometheus the series have `job="eod_reconciliation"`, not `job="pushgateway"`.

---

## Scenario 7: Scaling fleet

**Goal:** replace the hand-maintained market-data list with EC2 service discovery.

### Show the problem

1. ```bash
   scenarios/07-scaling-fleet/scale-fleet.sh status
   ```
   The fake AWS account has 3 running market-data instances, with tags.
2. **Prometheus → Status → Targets:** the `market-data` job has only `market-data-1`, the hand-written list.
3. **Grafana → TradeNova → Hosts and Fleet:** *Market-data servers discovered* = 1.

### Apply the fix

Show the new job in `scenarios/07-scaling-fleet/after/infra/prometheus/prometheus.yml`. Walk through it:
- `ec2_sd_configs` with `endpoint`, `region`, `filters` and `refresh_interval`;
- each line of `relabel_configs`;
- the comment explaining that in real AWS the `scrape_target` rule is removed and the private IP is used.

```bash
scripts/apply-scenario.sh 7
```

### What to point out in the result

- [ ] **Targets:** `market-data-1..3`, with labels `team`, `ec2_instance_id`, `instance_type` and `availability_zone`.
- [ ] **Status → Service Discovery → market-data:** the `__meta_ec2_*` labels before relabeling. Show how they become target labels.
- [ ] **Scale up, without touching Prometheus:**
  ```bash
  scenarios/07-scaling-fleet/scale-fleet.sh 5
  ```
  Within about 30 seconds the targets show 5 servers, and the Hosts and Fleet dashboard follows.
- [ ] **Scale down:** `scale-fleet.sh 2`, and the targets disappear. Then `scale-fleet.sh 3` to finish, because scenario 8 uses `market-data-2`.

---

## Scenario 8: On-call alerts

**Goal:** show that failures wake nobody, then load the alert rules and watch e-mails arrive.

### Show the problem

1. Break three things:
   ```bash
   scenarios/08-on-call-alerts/break-things.sh
   ```
   - a market-data server stops;
   - the batch job fails every run;
   - 30% of trade-api requests return HTTP 500.
2. Show the damage:
   - **Hosts and Fleet:** a target is down.
   - **Platform Overview:** *5xx error ratio* rises.
   - **Batch Jobs:** FAILED.
3. **Prometheus → Alerts:** "No alerting rules". **Mailpit:** empty. Nobody has been told.

### Apply the fix

Show the `rule_files` section added to `scenarios/08-on-call-alerts/after/infra/prometheus/prometheus.yml`. Then walk through:
- the three files in `infra/prometheus/rules/` (expressions, `for`, labels `severity` and `team`, annotations);
- `infra/alertmanager/alertmanager.yml` (grouping, the `team = "finance-ops"` route, e-mail receivers).

```bash
scripts/apply-scenario.sh 8
scenarios/08-on-call-alerts/break-things.sh      # again: applying the fix restarted the batch job in random mode
```

### What to point out in the result

- [ ] **Prometheus → Alerts:**
  - rules move from *Inactive* to **Pending** (the `for` duration), then **Firing**;
  - `EodReconciliationFailed` fires after 30 seconds;
  - `InstanceDown` and `ServiceHighErrorRate` fire after 1 minute.
- [ ] **Alertmanager UI:** alerts grouped by `alertname` and `job`.
- [ ] **Mailpit inbox:**
  - `[FIRING:1] EodReconciliationFailed ...` to **finance-ops@tradenova.example**;
  - `InstanceDown` and `ServiceHighErrorRate` to **platform-oncall@tradenova.example**.
- [ ] **Later (about 6 minutes after the last success):** `EodReconciliationStale` (critical) arrives.
- [ ] **Recovery:** run `scenarios/08-on-call-alerts/heal.sh`. Within a few minutes **[RESOLVED]** e-mails arrive (`send_resolved: true`).

---

## Scenario 9: Slow trades

**Goal:** find which hop is slow, then switch on Tempo's metrics generator for a service graph and RED metrics.

### Show the problem

1. Make trade-api slow:
   ```bash
   scenarios/09-slow-trades/inject-latency.sh 800
   ```
2. **Platform Overview:** *p95 latency* rises for both services. *Ask the room:* which one is the cause?
3. **Explore → Tempo → Service Graph tab:** empty, and **TradeNova → Services RED** is empty. Tempo stores traces but produces no metrics yet.
4. Traces still answer the question. **Explore → Tempo → TraceQL:**
   ```
   { resource.service.name = "portfolio-service" && duration > 500ms }
   ```
   Open a trace:
   - the `portfolio-service` server span is slow;
   - inside it, the `trade-api` span takes about 800 ms;
   - portfolio-service's own work is a few milliseconds.

### Apply the fix

Show the change in `scenarios/09-slow-trades/after/infra/tempo/tempo.yaml`:

```yaml
overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
```

Also show the `metrics_generator` block, which remote-writes to Prometheus.

```bash
scripts/apply-scenario.sh 9
```

Wait 1 to 2 minutes.

### What to point out in the result

- [ ] **Explore → Tempo → Service Graph:** `portfolio-service → trade-api`. Click the edge to see the request rate and latency. Click a node for RED panels.
- [ ] **TradeNova → Services RED:** request rate, error ratio, p95 by operation, and per-edge latency, all generated from traces.
- [ ] **TraceQL examples:**
  - `{ resource.service.name = "trade-api" && duration > 500ms }`: slow trade-api spans.
  - `{ status = error }`: failed spans.
  - `{ resource.service.name = "portfolio-service" } >> { duration > 500ms }`: portfolio requests with a slow span below them.
  - `{ resource.tradenova.team = "trading-core" }`: enrichment from scenario 1 at work.
- [ ] **Correlation:**
  - in a trace, open a trade-api span → **Logs for this span**, which shows Loki lines with the same `trace_id`;
  - in Loki, a log line → **Open trace in Tempo**.
- [ ] Clear the latency: `scenarios/09-slow-trades/inject-latency.sh 0`.

---

## After the session

- `scripts/reset-to-start.sh` puts every scenario back in its problem state, ready for the next run.
- `docker compose down` stops everything. `docker compose down -v` also deletes the stored data.
