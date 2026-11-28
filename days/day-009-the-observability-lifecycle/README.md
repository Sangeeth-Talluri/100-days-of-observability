# Day 9 — The Observability Lifecycle: Collect → Ship → Store → Query → Visualize → Alert

**In one line:** Telemetry travels a six-step pipeline — your app creates it, an agent ships it, a backend stores it, you query it, a dashboard shows it, and an alert wakes you when it goes wrong.

---

## 1. Explain it like I'm five

Imagine a hospital.

1. **Collect** — Nurses take your temperature, blood pressure, and heart rate. That is collecting.
2. **Ship** — The nurse writes the numbers on a chart and sends the chart to the records room. That is shipping.
3. **Store** — The records room files the chart in a cabinet, in order, so it can be found later. That is storing.
4. **Query** — A doctor asks, "Show me this patient's temperature for the last 24 hours." The records room finds the right charts. That is querying.
5. **Visualize** — The doctor looks at a chart with a line going up and down. One look shows the fever getting worse. That is visualizing.
6. **Alert** — A machine beeps when the heart rate drops too low. The nurse runs in before it gets worse. That is alerting.

If any step is missing, the hospital fails. Collect without storing? The numbers are lost. Store without querying? The data sits in a cabinet nobody opens. Visualize without alerting? Nobody watches the chart at 3 AM — the beep is what saves the patient.

Your system works the same way. Observability is not one tool. It is this six-step lifecycle, and every step matters.

---

## 2. The real definition

The observability lifecycle is the path telemetry takes from your code to a human decision. Six stages, in order:

| Stage | Plain meaning | Example tools |
|---|---|---|
| **1. Collect** | Your code or an agent creates telemetry | OpenTelemetry SDKs, Prometheus exporters, log files |
| **2. Ship** | The telemetry is sent over the network to a backend | OpenTelemetry Collector, Promtail, Fluent Bit, Grafana Alloy |
| **3. Store** | A backend keeps it, indexed, for days or months | Prometheus, Loki, Tempo, Jaeger, Elasticsearch |
| **4. Query** | You ask questions in a query language | PromQL, LogQL, TraceQL, KQL |
| **5. Visualize** | Charts and dashboards turn answers into pictures | Grafana, Kibana, cloud consoles |
| **6. Alert** | Rules watch the data and page a human when it breaks a limit | Alertmanager, Grafana Alerting, PagerDuty |

Two important facts:

- **Each stage is a separate choice.** You can collect with OpenTelemetry, ship with the Collector, store in Prometheus, and visualize in Grafana — four different tools, one pipeline. That is normal. This is why Day 6 (OpenTelemetry) matters: a shared standard lets the stages mix and match.
- **The pipeline only flows one way, but the value flows back.** An alert (stage 6) sends you to a dashboard (stage 5), which runs a query (stage 4) against stored data (stage 3). The incident response you learned on Day 1 is really a trip *backwards* through this pipeline.

---

## 3. A real example

Your checkout service runs in production. Here is one slow request, traveling the whole lifecycle:

1. **Collect** — The checkout code is instrumented with the OpenTelemetry SDK (Day 7). Every request creates a trace with spans: `checkout`, `charge-card`, `send-email`. The app also exposes a Prometheus metrics endpoint: `http_requests_total`, `http_request_duration_seconds`. Its logs go to stdout as JSON.
2. **Ship** — The OpenTelemetry Collector runs next to the app. It receives traces and metrics on OTLP port 4317, and Promtail (or Grafana Alloy) tails the log files and ships them to Loki. Nothing is stored on the app machine — the app just hands data off and forgets it.
3. **Store** — Prometheus scrapes and stores the metrics for 15 days. Tempo stores the traces. Loki stores the logs. Each backend is built for one signal — metrics backends are fast at numbers, log backends are good at text, trace backends are good at request trees. (Days 13, 29, and 43 go deep on each.)
4. **Query** — A slow checkout is reported. You open Grafana Explore and run `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` — the 99th-percentile latency. It is 8 seconds, and only for the `charge-card` span. You run a LogQL query `{app="checkout"} |= "error"` and see `timeout after 3 retries` for one payment provider.
5. **Visualize** — Your checkout dashboard (Day 52) already shows latency, error rate, and traffic on one screen. The red spike at 3 PM lines up exactly with the payment provider's outage window.
6. **Alert** — You already have a rule: "page me if p99 checkout latency is above 2 seconds for 5 minutes." It fired at 3:02 PM — before the first customer complaint. The alert message links straight to the dashboard.

One request. Six stages. Without *any* one of them, the story breaks: no collect, no data exists; no ship, data never leaves the app; no store, yesterday's data is gone; no query, you cannot ask; no visualize, you cannot see the pattern; no alert, you find out from angry customers.

---

## 4. What problem does it solve?

Beginners usually buy **one** tool and expect observability. The lifecycle explains why that fails:

- **"We installed Prometheus, but we still can't debug."** Prometheus is stage 3 (store) and stage 4 (query) for *metrics only*. Traces and logs are different stages and different backends. One tool is one or two stages, never all six.
- **"We have logs, but we can't find anything."** You collected and stored, but your logs have no structure, no request ID, and no good query. Stage 4 is broken, so stage 3 is a graveyard.
- **"We found out from Twitter."** Your dashboards were fine, but nobody watches dashboards at 3 AM. Stage 6 — alerting — was missing.
- **"The data stops during the incident."** Your shipping stage had no buffer. When the backend went down, or the network choked, telemetry was lost exactly when you needed it most. (Day 84 covers buffering and backpressure.)

The lifecycle gives you a **checklist for every incident**: walk the six stages and find which one broke. That is faster than guessing, and it tells you exactly what to fix.

---

## 5. When to use it — and when NOT to

**Use the full lifecycle when:**
- You run services in production that other people depend on.
- An incident costs money, users, or sleep.
- You need to answer questions *after* the fact — which means data must be stored and queryable, not just watched live.

**A shorter pipeline is fine when:**
- A side project or a local dev environment — a log file on disk and `tail -f` can be enough. You still have collect → store → query, just in the simplest form.
- A tiny single-server app — you may skip the shipping stage entirely and read metrics straight from the app. Add shipping when you have more than one machine.

**Honest trade-off:** every stage you add costs something — CPU on your app, network traffic, storage money, and one more thing to operate. The lifecycle is not "add all six stages at maximum power." It is "know which stage is missing when you are blind, and add the cheapest version of it." (Day 83 covers cost control in detail.)

---

## 6. Hands-on: build the whole lifecycle on your laptop (about 30 minutes)

You will run the full pipeline with Docker Compose: an app sends telemetry to the OpenTelemetry Collector, which ships metrics to Prometheus, logs to Loki, and traces to Tempo — and Grafana visualizes and alerts on all of it.

### Step 1 — The Collector config (stages 1–2: collect and ship)

Create `otelcol-config.yaml`. This is the heart of the lifecycle: receivers take data in, processors clean it, exporters send it out. The `service.pipelines` section wires them together.

```yaml
# otelcol-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 75
    spike_limit_percentage: 15

exporters:
  debug:
    verbosity: detailed
  prometheus:
    endpoint: 0.0.0.0:8889
  otlp:
    endpoint: tempo:4317
    tls:
      insecure: true
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, otlp]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, prometheus]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, loki]
```

Key idea to notice: **one receiver feeds three pipelines.** The app sends everything to port 4317 once, and the Collector splits it into traces, metrics, and logs. That is the shipping stage doing its job.

Validate the config before you run it (catches typos early):

```bash
docker run --rm -v $(pwd)/otelcol-config.yaml:/etc/otelcol-contrib/config.yaml \
  otel/otelcol-contrib:latest validate --config=/etc/otelcol-contrib/config.yaml
```

Expected output: no errors — just a quiet confirmation that the config is valid. If it complains, fix the line it names and re-run.

### Step 2 — The whole stack (stages 3–5: store and visualize)

Create `docker-compose.yaml`:

```yaml
services:
  otelcol:
    image: otel/otelcol-contrib:latest
    command: ["--config=/etc/otelcol-contrib/config.yaml"]
    volumes:
      - ./otelcol-config.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC — apps send telemetry here
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Collector's Prometheus metrics

  prometheus:
    image: prom/prometheus:latest
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  loki:
    image: grafana/loki:latest
    command: ["-config.file=/etc/loki/local-config.yaml"]
    ports:
      - "3100:3100"

  tempo:
    image: grafana/tempo:latest
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
    ports:
      - "3200:3200"

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    ports:
      - "3000:3000"
```

Prometheus needs to scrape the Collector's metrics endpoint — create `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "otelcol"
    static_configs:
      - targets: ["otelcol:8889"]
```

Tempo needs a minimal config — create `tempo.yaml`:

```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317

storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/traces
```

Start it all:

```bash
docker compose up -d
docker compose ps
```

Expected output: all five containers `Up` (or `running`). If one restarts in a loop, run `docker compose logs <name>` — usually a wrong port or a missing config file.

### Step 3 — Send some telemetry (stage 1: collect)

Any OTel-instrumented app works. The fastest test is a single `curl` sending a log and a metric-shaped trace is overkill — instead, use the Collector's own debug output: it logs every span it receives. Send a test trace with the `telemetrygen` tool:

```bash
docker run --rm --network host \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  traces --otlp-endpoint localhost:4317 --otlp-insecure --traces 5
```

Then check the Collector saw them:

```bash
docker compose logs otelcol | grep -i "span" | head -5
```

Expected output: lines showing received spans (names like `operation-...`). If you see nothing, check that port 4317 is listening: `curl localhost:13133/` should return the Collector health page (the `health_check` extension is on by default in the contrib image at port 13133 only if enabled — if it 404s, check the logs instead).

### Step 4 — Query each stage (stage 4: query)

Open Prometheus at <http://localhost:9090> and run:

```promql
otelcol_receiver_accepted_spans_total
```

You should see a number that grows — those are the spans the Collector accepted. That is stage 4 working on stage 3's stored data.

Open Grafana at <http://localhost:3000>, add Loki as a data source (`http://loki:3100`), and run this LogQL query in Explore:

```logql
{service_name="telemetrygen"} |= "error"
```

Then the metric version of the same question — error rate per second:

```logql
sum(rate({service_name="telemetrygen"} |= "error" [5m]))
```

No errors in the test data? Good — that is the *correct* answer (zero). Now change `error` to a word that does appear, and watch the rate become non-zero. The query language is the same shape whether you ask about logs or metrics; only the backend changes.

### Step 5 — Alert (stage 6: alert)

In Grafana, go to **Alerting → Alert rules → New alert rule** and use the LogQL query from Step 4:

```logql
sum(rate({service_name="telemetrygen"} |= "error" [5m]))
```

- **Reduce:** Last
- **Threshold:** Is above `10`
- **Evaluate:** every 1 minute, pending for 5 minutes

Save it. You now have all six stages running on your laptop. When you are done experimenting, tear it down with `docker compose down`.

### What to look for

- **Each stage has its own port.** 4317 (collect), 8889 (Collector metrics), 9090 (Prometheus), 3100 (Loki), 3200 (Tempo), 3000 (Grafana). If a stage is broken, its port tells you where to look.
- **The debug exporter is your friend.** While learning, `verbosity: detailed` shows exactly what flows through the Collector. Turn it off in production — it is slow and noisy.
- **The Collector is the shipping stage, not the storage stage.** It holds data in memory for seconds, then forwards it. If it crashes, in-flight data is lost unless you add a buffer (Day 84).

---

## 7. Key terms from today

- **Pipeline** — the path telemetry takes from app to alert: collect → ship → store → query → visualize → alert.
- **Receiver** — the Collector component that takes telemetry in (OTLP, Prometheus scrape, Jaeger, Kafka).
- **Processor** — the Collector component that changes telemetry in flight: batching, sampling, adding labels, dropping noisy data.
- **Exporter** — the Collector component that sends telemetry out to a backend.
- **Backend** — the system that stores one signal: Prometheus (metrics), Loki (logs), Tempo/Jaeger (traces).
- **Scrape** — Prometheus pulling metrics from an app on a schedule, instead of the app pushing them.
- **Retention** — how long a backend keeps data before deleting it. Longer retention costs more storage.

---

## 8. Common mistakes

1. **Buying a dashboard tool and calling it observability.** A dashboard is stage 5. Without collect, ship, store, and query underneath, it shows empty panels. Start at stage 1, not stage 5.
2. **Skipping the shipping stage and letting every app talk to every backend.** It works with 2 services and collapses at 20 — every backend change means redeploying every app. A Collector (or agent) in the middle means apps send to one place, and you change the backend in one config file.
3. **No buffering in the shipping stage.** If the backend goes down for 10 minutes, unbuffered shippers drop data — exactly the data you need to explain the outage. Even a small disk buffer or retry queue saves you. (Day 84 goes deep.)

---

## 9. Recap — the 5 things to remember

1. Observability is a six-stage pipeline: collect → ship → store → query → visualize → alert.
2. Each stage is a separate choice — you can mix tools, and OpenTelemetry is the standard that lets them fit together.
3. One missing stage breaks the whole story: no alert means you hear it from customers; no store means yesterday is gone.
4. The OpenTelemetry Collector is the shipping stage: receivers in, processors in the middle, exporters out, wired together by pipelines.
5. When an incident confuses you, walk the six stages and find which one broke — that tells you exactly what to fix.

**Tomorrow — Day 10:** Cardinality: the one concept that decides your power and your bill — why one bad label can cost you thousands.
