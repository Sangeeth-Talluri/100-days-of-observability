# Day 6 — OpenTelemetry: The Vendor-Neutral Standard

**In one line:** OpenTelemetry is the one open standard for getting traces, metrics, and logs out of your code — you instrument once, and you can send the data to any backend you want, without rewriting your code.

---

## 1. Explain it like I'm five

Imagine every country has its own kind of electrical plug. Your phone charger works in your country, but when you travel, you need a new charger — or a bag full of adapters.

Before OpenTelemetry, observability worked like that. Every vendor had its own agent, its own library, its own format:

- Your checkout service used **Vendor A's** agent to send traces.
- Your search service used **Vendor B's** library to send metrics.
- Your billing service used **Vendor C's** agent to send logs.

If you wanted to switch vendors — because of price, or because one vendor was better at traces and another better at logs — you had to rewrite the code in every service. That is expensive, slow, and scary. So most teams never switched, even when they were unhappy.

OpenTelemetry is the **universal plug**. It is one open standard that says: "This is how you create a span. This is how you record a metric. This is how you send a log. This is the format the data travels in."

You write your code against the OpenTelemetry API once. Then a small program called the **Collector** takes your data and sends it wherever you want — Grafana, Datadog, Honeycomb, Jaeger, your own storage, two backends at the same time. Switching backends becomes changing one line in a config file, not rewriting your services.

That is why the whole industry agreed on it. In May 2026, the Cloud Native Computing Foundation (CNCF) moved OpenTelemetry to **Graduated** status — its highest maturity level, the same level as Kubernetes and Prometheus. It is now the second most active project in the CNCF, with more than 12,000 contributors from over 2,800 companies. Every major observability vendor accepts its data.

---

## 2. The real definition

OpenTelemetry — **OTel** for short — is not one program. It is a set of open pieces that work together:

| Piece | What it is | What it does |
|---|---|---|
| **API** | A standard set of calls your code uses | `start a span`, `record a metric`, `emit a log` — the same calls in every language |
| **SDK** | The code that runs the API in your app | Collects what the API records, adds context (which service, which version), and sends it on |
| **OTLP** | The OpenTelemetry Protocol | The wire format: how telemetry travels from your app to the Collector or backend, over gRPC (port 4317) or HTTP (port 4318) |
| **Semantic conventions** | Standard names for common fields | `service.name`, `http.route`, `http.response.status_code` — so every backend reads the same fields the same way |
| **Collector** | A small standalone program | Receives telemetry, processes it (filter, sample, add labels), and exports it to one or more backends |
| **Auto-instrumentation** | Libraries that patch your frameworks | Create spans for HTTP calls, database queries, and queues automatically — no code changes |

### The two ideas that make it work

**1. API and SDK are separate.** Your code only ever talks to the API. The API is stable and vendor-neutral — it has no vendor's name in it. The SDK underneath decides where the data goes. That split is what makes switching backends a config change instead of a code change.

**2. One pipeline, three signals.** The same API, the same protocol, and the same Collector carry traces, metrics, and logs. You do not learn three tools and run three agents. You learn one standard, and it covers all three pillars from Day 4.

### How mature is each part?

Not every part is equally finished. As of late 2026, the official status is:

| Signal | API | SDK | OTLP |
|---|---|---|---|
| **Traces** | Stable | Stable | Stable |
| **Metrics** | Stable | Stable | Stable |
| **Logs** | Stable (bridge API) | Stable | Stable |
| **Profiles** | — | — | Alpha (public alpha since March 2026) |

One warning: the *specification* being stable does not mean every *language* is equally ready. Java and .NET are stable for traces, metrics, and logs. Python is stable for traces and metrics, while its logs support is still maturing. Go's logs are a release candidate. Before you adopt a signal in a language, check that language's status on opentelemetry.io — Day 7 goes into the per-language details.

### A short history (so the names make sense)

OpenTelemetry was born in 2019 from a merger of two earlier projects: **OpenTracing** (from Uber) and **OpenCensus** (from Google). Both tried to solve the same problem, and the industry was tired of choosing between them — so they merged into one project. OpenTracing is now archived; OpenCensus is in maintenance mode. OpenTelemetry is the standard, and in May 2026 the CNCF graduated it.

### The one thing OpenTelemetry is NOT

OpenTelemetry is **not a backend**. It does not store your data, and it does not draw your graphs. It collects and ships telemetry. You still need a backend to receive it — Jaeger or Tempo for traces, Prometheus or Mimir for metrics, Loki or Elasticsearch for logs, Grafana to see it all. OTel is the road; the backend is the destination.

---

## 3. A real example

Your company runs three services: `checkout`, `search`, and `billing`. Each one was built by a different team, in a different year, with a different vendor's agent:

- `checkout` (Python) sends traces to Vendor A with Vendor A's library.
- `search` (Go) sends metrics to Vendor B with Vendor B's agent.
- `billing` (Java) sends logs to Vendor C with Vendor C's agent.

Then two things happen in the same month:

1. **Vendor A raises its price by 40%.** You want to move traces to a cheaper backend. That means rewriting the tracing code in `checkout` — and re-testing every endpoint — because the tracing calls are Vendor A's calls.
2. **An incident hits all three services.** A checkout request calls `search`, which calls `billing`. But the trace stops at each service boundary: each vendor's agent writes its own trace ID in its own format, so no single tool can follow one request end to end. You debug with three tabs open and a lot of guessing.

Now the same company, after moving to OpenTelemetry:

- All three services use the **OTel API** in their own language. The code says `start a span` — it never says a vendor's name.
- All three services send their telemetry to one **OTel Collector** running in the cluster.
- The Collector fans the data out to **two backends at once**: Grafana Cloud for day-to-day work, and Vendor A for the one team that still likes it.
- When Vendor A raises its price again, the team changes **one exporter line in the Collector config** and Vendor A is gone. No service is touched. No code is rewritten.
- During the next incident, one request has **one trace ID** from `checkout` through `search` to `billing`, because every service speaks the same protocol and passes the same W3C `traceparent` header. One trace, end to end, in one tool.

The code did not get smarter. The *plumbing* got standard. That is the whole point of OpenTelemetry: boring, shared plumbing, so your energy goes into fixing systems instead of rewiring them.

---

## 4. What problem does it solve?

**Vendor lock-in.** Before OTel, your instrumentation code was married to your vendor. Switching vendors meant rewriting code in every service — so teams stayed with vendors they had outgrown. OTel breaks that marriage: the code is neutral, and the backend is a config choice.

**The three-agents problem.** Before OTel, a typical service ran a tracing agent, a metrics agent, and a log shipper — three programs, three configs, three things to update and secure. OTel gives you one SDK in the app and one Collector outside it, for all three signals.

**Polyglot chaos.** In a company with Python, Go, Java, and Node.js services, each language had its own vendor libraries with different features and different bugs. OTel gives every language the same API, the same protocol, and the same naming rules — so a trace that starts in Python and ends in Go looks the same all the way through.

**Incompatible data.** Even when two vendors both claimed to "support tracing," their span formats differed, so traces could not cross service boundaries. OTel's shared protocol (OTLP) and shared naming rules (semantic conventions) mean the data means the same thing everywhere it lands.

---

## 5. When to use it — and when NOT to

**Use OpenTelemetry when:**
- You run more than one service, or you expect to. One standard now saves a rewrite later.
- You might change backends one day — for price, features, or a company decision. With OTel, that change is a config edit, not a project.
- You run several languages. One standard across Python, Go, Java, and Node.js beats four vendor libraries.
- You want traces, metrics, and logs to share one pipeline and one set of labels, so they correlate (Day 36 goes deep on correlation).

**Do NOT use OpenTelemetry when:**
- The thing you are building is a small script or a one-off job, and a single log line is enough. OTel is plumbing — don't lay pipes for a cup of water.
- The signal you need is not stable yet in your language. Profiles are alpha everywhere; Python logs are still maturing. Check the status table in Section 2 first — using an alpha signal in production is a bet, not a default.
- You need a deep vendor-specific feature that the generic exporter cannot express (for example, a vendor's special profiling or security product). In that case, use the vendor's SDK *alongside* OTel for that one feature — not *instead of* OTel for everything.

**Honest rule:** for any service that will live longer than a few months, instrument with OpenTelemetry from day one. Adding it later means touching code you have already forgotten.

---

## 6. Hands-on: run the Collector, instrument a Python service, and watch the data flow (45 minutes)

You need **Docker** and **Python 3.10+**. Everything runs on your own machine — no cloud account, no vendor signup. The Collector prints what it receives to its own log, so it is also its own backend for this exercise.

### Part A — Run the OpenTelemetry Collector in Docker

**Step 1 — Save this as `otelcol.yaml`.**

This config receives telemetry over OTLP (gRPC on 4317, HTTP on 4318), batches it, and prints it with the `debug` exporter. The `health_check` extension lets you check the Collector is alive.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:

exporters:
  debug:
    verbosity: detailed

extensions:
  health_check:

service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

**Step 2 — Check the config is valid before you run it.**

```bash
docker run --rm \
  -v "$(pwd)/otelcol.yaml:/etc/otelcol-contrib/config.yaml" \
  otel/opentelemetry-collector-contrib:0.164.0 validate --config=/etc/otelcol-contrib/config.yaml
```

A valid config exits with code 0. (Pin the exact version — never use `:latest` for the Collector: it is still a `0.x` program, and releases can rename components. In version 0.144.0 the `otlp` exporter was renamed `otlp_grpc` and `otlphttp` became `otlp_http` — old tutorials still show the old names.)

**Step 3 — Start the Collector.**

```bash
docker run --rm --name otel-collector \
  -p 4317:4317 -p 4318:4318 -p 13133:13133 \
  -v "$(pwd)/otelcol.yaml:/etc/otelcol-contrib/config.yaml" \
  otel/opentelemetry-collector-contrib:0.164.0
```

Expected output (trimmed) — the Collector starts all three pipelines:

```text
info    service@v0.164.0/service.go:...  Starting otelcol-contrib...
info    service@v0.164.0/service.go:...  Applying configuration
info    builder/pipelines.go:...         Starting receivers...
info    builder/pipelines.go:...         Starting processors...
info    builder/pipelines.go:...         Starting exporters...
info    service@v0.164.0/service.go:...  Everything is ready. Begin running and processing data.
```

**Step 4 — Check it is healthy.**

```bash
curl -s http://localhost:13133/
```

Expected output:

```json
{"status":"Server available","upSince":"2026-11-25T08:10:00Z"}
```

### Part B — Auto-instrument a small Flask service (zero code changes)

**Step 1 — Save this as `app.py`.** A tiny checkout service:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/checkout", methods=["POST"])
def checkout():
    return {"status": "paid", "order_id": "ord-1234"}, 200

@app.route("/health")
def health():
    return {"status": "ok"}, 200

if __name__ == "__main__":
    app.run(port=5000)
```

**Step 2 — Install the OTel distro and the OTLP exporter.**

```bash
pip install flask opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

`opentelemetry-bootstrap -a install` scans your installed packages and installs the matching instrumentation libraries — because `flask` is installed, it adds the Flask instrumentation for you.

**Step 3 — Run the app through the auto-instrumentation launcher.**

```bash
OTEL_SERVICE_NAME=checkout \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
opentelemetry-instrument python app.py
```

Two environment variables do all the configuration:

- `OTEL_SERVICE_NAME` — the name your service shows up as. Always set it; without it, everything is called `unknown_service`.
- `OTEL_EXPORTER_OTLP_ENDPOINT` — where to send the data. The default OTLP protocol for the Python exporter is gRPC, so this points at the Collector's gRPC port, 4317. (If you use HTTP instead, set `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf` and point at port 4318 — the exporter appends `/v1/traces`, `/v1/metrics`, and `/v1/logs` itself.)

**Step 4 — Send a request and watch the Collector print the trace.**

```bash
curl -s -X POST http://localhost:5000/checkout
```

In the Collector's terminal you will see a span arrive (trimmed):

```text
ResourceSpans #0
Resource SchemaURL:
Resource attributes:
     -> service.name: Str(checkout)
ScopeSpans #0
InstrumentationScope flask
Span #0
    Trace ID       : 9f1c2a7b4e5d6a8b9c0d1e2f3a4b5c6d
    Parent ID      :
    Name           : POST /checkout
    Kind           : Server
    Start time     : 2026-11-25 08:12:00 +0000 UTC
    End time       : 2026-11-25 08:12:00.003 +0000 UTC
    Status code    : Unset
    Attributes:
         -> http.request.method: Str(POST)
         -> http.route: Str(/checkout)
         -> http.response.status_code: Int(200)
```

You wrote **no tracing code**. The Flask instrumentation created the span, named it from the method and route, and filled in the standard attributes from the semantic conventions. That is auto-instrumentation: free traces for every request.

### Part C — Add one manual span and one manual metric

Auto-instrumentation covers the framework. Your *business logic* — the payment call inside checkout — needs a few lines of your own code. Add this to `app.py`:

```python
from opentelemetry import trace, metrics

tracer = trace.get_tracer("checkout.tracer")
meter = metrics.get_meter("checkout.meter")
orders_counter = meter.create_counter(
    "checkout.orders.total",
    description="Number of completed checkouts",
)

@app.route("/checkout", methods=["POST"])
def checkout():
    with tracer.start_as_current_span("charge-card") as span:
        span.set_attribute("payment.provider", "card-provider-b")
        # ... call the payment provider here ...
        span.add_event("payment.retry", {"attempt": 2})
        orders_counter.add(1, {"payment.method": "card"})
    return {"status": "paid", "order_id": "ord-1234"}, 200
```

Restart the app the same way as in Part B, Step 3, send another `POST /checkout`, and look at the Collector output. You will now see two spans for one request:

- `POST /checkout` — the auto-instrumented server span (the parent).
- `charge-card` — your manual span (the child), carrying the `payment.provider` attribute and the `payment.retry` event.

And in the metrics pipeline, a counter called `checkout.orders.total` with the label `payment.method=card`, going up by 1 on every checkout. Three signals, one API, one Collector, zero vendor code.

**Try it yourself:** change the exporter in `otelcol.yaml` from `debug` to a real backend — for example, point `otlp_http` at a local Jaeger or Tempo — and notice that `app.py` does not change at all. That is the whole promise of OpenTelemetry: the code stays the same, only the config moves.

---

## 7. Key terms from today

- **OpenTelemetry (OTel)** — the open, vendor-neutral standard for generating, collecting, and exporting traces, metrics, and logs. A graduated CNCF project since May 2026.
- **API** — the standard calls your code uses (`start a span`, `record a metric`). Vendor-neutral and stable; your code only ever talks to this.
- **SDK** — the code that implements the API in your app: it collects what the API records, adds resource context, and sends it on.
- **OTLP (OpenTelemetry Protocol)** — the wire protocol that carries telemetry from SDKs to Collectors and backends, over gRPC (port 4317) or HTTP (port 4318).
- **Semantic conventions** — the standard names for common attributes (`service.name`, `http.route`, `http.response.status_code`), so every backend reads the same fields the same way.
- **Collector** — a standalone program that receives telemetry, processes it (batching, filtering, sampling, adding labels), and exports it to one or more backends. Distributions: core (`otelcol`), contrib (`otelcol-contrib`, with hundreds of community components), Kubernetes, and vendor builds.
- **Auto-instrumentation** — libraries that patch your frameworks at startup so you get traces and metrics without changing your code (for example, `opentelemetry-instrument` for Python, the Java agent JAR).
- **Manual instrumentation** — spans, metrics, and logs you create yourself in code, for your business logic.
- **Resource** — the identity of the thing emitting telemetry: `service.name`, `service.version`, `deployment.environment`. Always set `service.name`.
- **Exporter** — the Collector component that sends data to a backend (`debug`, `otlp_http`, `otlp_grpc`, and hundreds of vendor-specific ones).
- **W3C traceparent** — the standard HTTP header that carries the trace ID across service boundaries, so one trace can span many services.
- **Vendor lock-in** — when switching tools means rewriting code. OTel exists to end it.

---

## 8. Common mistakes

1. **Thinking OpenTelemetry is a backend.** It collects and ships data — it does not store it or draw graphs. If you install the Collector and wonder where the dashboards are, you are missing the backend (Jaeger, Tempo, Prometheus, Grafana). OTel is the road, not the destination.
2. **Hard-coding a vendor's exporter in your code.** If your code imports a vendor's exporter, you have rebuilt the lock-in OTel was meant to remove. Keep vendor details in the Collector config; keep your code on the OTel API.
3. **Leaving `service.name` unset.** Everything shows up as `unknown_service`, and in a system with twenty services you cannot tell them apart. Set `OTEL_SERVICE_NAME` (or `service.name` in the resource) for every service, from day one.
4. **Copying old tutorials with renamed components.** The Collector moves fast — in v0.144.0 the `otlp` exporter became `otlp_grpc` and `otlphttp` became `otlp_http`. Pin your Collector version, check the changelog before upgrading, and validate your config with `otelcol-contrib validate` before deploying.
5. **Putting high-cardinality data in span attributes.** A user ID, an email, or a raw URL with a query string as an attribute creates a new unique value for every request — exploding storage and cost, and possibly leaking personal data. Attributes are for low-cardinality facts (`payment.provider`, `http.route`), not for identifiers. (Day 10, on cardinality, goes deep.)

---

## 9. Recap — the 5 things to remember

1. OpenTelemetry is the vendor-neutral standard for traces, metrics, and logs — one API, one protocol (OTLP), one Collector — and it graduated in the CNCF in May 2026.
2. Your code talks only to the OTel API; the SDK and the Collector decide where data goes. Switching backends is a config change, not a rewrite.
3. The Collector is the hub: it receives OTLP, processes the data (batch, filter, sample), and exports it to any backend — or several at once.
4. Auto-instrumentation gives you free spans for frameworks (HTTP, databases, queues); manual instrumentation covers your business logic — one span, one metric, a few lines of code.
5. OTel is not a backend: it collects and ships. You still need Jaeger, Tempo, Prometheus, Loki, or Grafana to store and see the data — and always set `service.name`.

**Tomorrow — Day 7:** Instrumentation: manual vs automatic — when the free auto-instrumentation is enough, when you need to write your own spans, and how to mix the two without making a mess.

---

*Sources checked for this lesson: OpenTelemetry project status and CNCF graduation (May 2026); Collector releases v0.161.0–v0.164.0 and the v0.144.0 exporter rename; Python SDK auto-instrumentation docs (`opentelemetry-distro`, `opentelemetry-bootstrap`, `opentelemetry-instrument`); OTLP default ports 4317 (gRPC) and 4318 (HTTP). Versions were current as of November 2026 — pin exact versions and re-check the changelog before upgrading.*
