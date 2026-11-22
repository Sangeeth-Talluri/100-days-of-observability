# Day 3 — Telemetry: The Data Observability Runs On

**In one line:** Telemetry is the raw data your system sends out about itself — every number, log line, and trace — and without it, there is nothing to observe.

---

## 1. Explain it like I'm five

Imagine a Formula 1 race car.

The driver sits inside the car. He can feel the steering wheel and hear the engine, but he cannot see the engine temperature, the tire pressure, or how much fuel is left. So the car has **sensors** — small devices that measure things — and a **radio** that sends those numbers to the pit crew every second.

The pit crew sits far away, watching screens. They see: speed 310 km/h, engine 118°C, tire pressure falling on the left rear wheel. They radio back: "Come in now, your tire is going flat." The driver never saw the tire. The data saved the race.

That is telemetry. The word comes from Greek: **tele** (far away) + **metron** (measure) — *measuring from far away*. It was first used for rockets and spacecraft: NASA could not ride along with a rocket, so the rocket had to send data home. Your system is the race car. Telemetry is the radio link that lets you see inside it while it runs.

No radio, no data. No data, no observability. Everything in this series — every dashboard, every alert, every trace — is built from telemetry.

---

## 2. The real definition: three parts in every piece of telemetry

**Telemetry is any data a system emits about its own behavior while it runs.** It is the raw material. Observability is what you *do* with it: collect it, store it, query it, and ask questions of it.

Every single piece of telemetry — a log line, a metric sample, a trace span — has the same three parts:

| Part | What it answers | Example |
|---|---|---|
| **What happened** (the signal) | The event itself | "Checkout took 342 ms" / "Order placed" |
| **When** (the timestamp) | The exact moment | `2026-11-22T13:05:26Z` |
| **Context** (the attributes) | Who, where, which | `service=checkout`, `region=eu-west`, `user_id=user-4821` |

The context is what makes telemetry powerful. A number without context is just a number. A number with `region`, `provider`, and `version` attached is a number you can **slice** — and slicing is how Day 2's hidden failure was found.

### The signals (a preview — Day 4 goes deep on each)

- **Metrics** — numbers over time: how many, how slow, how full. ("Checkout latency is 342 ms.")
- **Logs** — events in words: what happened, in order. ("Order placed for user-4821.")
- **Traces** — one request's journey across services: which steps it took, and how long each step lasted.
- **Events and profiles** — extra signals, like "a deploy happened" or "this function used 80% of CPU." (Day 5.)

### Where telemetry comes from

| Source | What it emits | Example |
|---|---|---|
| **Your code** (instrumentation) | Business and request data | A checkout handler records latency and result |
| **The language runtime** | Garbage collection, thread pools, memory | JVM heap usage, Go goroutine count |
| **The operating system and kernel** | CPU, disk, network, processes | `node_exporter` reads `/proc` on Linux |
| **The platform** | Kubernetes, cloud services | Pod restarts, load balancer errors, Lambda cold starts |

### The journey of a piece of telemetry

Once emitted, telemetry travels a pipeline — the same one every day of this series touches:

```
emit → collect → process → store → query → visualize / alert
```

- **Emit:** your code or an agent produces the data.
- **Collect:** something gathers it — an agent, a Collector, a scrape. (Days 45, 13.)
- **Process:** filter out noise, add labels, sample the flood down to a stream. (Days 35, 41.)
- **Store:** a backend keeps it — Prometheus, Loki, Tempo, or a cloud service.
- **Query:** you ask questions — PromQL, LogQL, KQL. (Days 14, 31, 68.)
- **Visualize / alert:** dashboards and alerts turn answers into action. (Days 51–59.)

If any link in this chain breaks, the data never reaches you — and during an incident, that is the same as having no data at all. Day 9 walks this whole lifecycle in detail.

### One standard moves it all: OTLP

Different tools used to speak different languages, so every vendor needed its own agent. OpenTelemetry fixed that with **OTLP** — the OpenTelemetry Protocol — one standard way to send metrics, logs, and traces. An app sends OTLP to a Collector (or straight to a backend) on port **4317** (gRPC) or **4318** (HTTP). Day 6 covers OpenTelemetry itself; Day 46 covers OTLP in depth. For today, just remember the two ports — you will use them in the hands-on section.

---

## 3. A real example: one slow checkout, told three ways

It is Tuesday, 14:02. Your on-call phone buzzes: checkout is slow. Three signals arrive, each telling part of the story:

**The metric speaks first.** Your latency graph shows checkout p99 jumped from 300 ms to 8 seconds — but only for `region=eu-west`. The graph says *where* and *how bad*. It cannot say *why*.

**The log speaks second.** You search the checkout logs for that minute and find:

```json
{"timestamp": "2026-11-22T14:02:17Z", "level": "ERROR", "service": "checkout", "message": "payment timeout after 5000ms", "provider": "provider-b", "trace_id": "9f31ac..."}
```

Now you know the *what*: the payment call timed out, with provider-b. But which exact request, and which step inside it?

**The trace speaks third.** You paste the `trace_id` into your tracing tool. One request, 8.2 seconds total:

```
POST /checkout                     8.2s
├── validate cart                  0.1s
├── charge payment (provider-b)    7.9s   ← here
└── send confirmation email        0.2s
```

The trace shows the *exact step*: the payment call ate 7.9 of the 8.2 seconds.

None of the three signals alone solves the incident. The metric found it, the log named it, the trace proved it. Together, the answer took **4 minutes**. Without telemetry, the same incident is a meeting, a guess, and a rollback of the wrong service.

Notice the quiet hero of this story: the **`trace_id`**. The same ID sits in the log line and in the trace. That shared ID is what lets you jump from one signal to another. Telemetry without shared IDs is three separate puzzles. Telemetry with shared IDs is one story.

---

## 4. What problem does it solve?

1. **No data, no answers.** Observability is not magic — it is questions asked of data you already collected. Telemetry is that data. If your system emits nothing, there is nothing to query during an incident, no matter how good your tools are.
2. **Evidence before the incident.** You cannot add telemetry after the crash. The failure you will debug next month can only be answered by data you collect *now*. Telemetry is preparation, paid in advance.
3. **One story across signals.** A shared `trace_id` ties a log line to a trace span to a metric spike. Without that link, each signal is a clue in a different room.
4. **Facts instead of opinions.** "I think the database is slow" becomes "the trace shows the database step took 7.9 seconds." Incidents get shorter when the argument ends and the evidence starts.
5. **A record of normal.** Telemetry does not just help in incidents. Months of it show you what "normal" looks like — so you can spot drift, plan capacity (Day 82), and prove a fix actually worked.

---

## 5. When to collect a lot — and when to hold back

Telemetry is not free. Every byte costs CPU to emit, network to move, and money to store. Good teams collect *enough*, not *everything*.

**Collect richly when:**
- The service is user-facing or on the money path — checkout, login, payments. Blind spots here cost real money.
- Failures are expensive or hard to reproduce — production-only bugs, rare combinations (Day 2).
- You are learning a new system — early on, more data teaches you what "normal" looks like.
- The data answers a question you actually ask — latency by region, errors by provider.

**Hold back when:**
- The data is **high-cardinality** — a label with millions of values, like a user ID or an email address, attached to a metric. It explodes storage and cost. Day 10 explains this in full.
- It contains **secrets or personal data** — passwords, API keys, credit card numbers, full user details. Never put these in logs or spans. Ever.
- It is noise you never query — DEBUG logs in production, a metric nobody has looked at in six months. Delete it or sample it down.
- The cost outweighs the value — storing every byte of a chatty service for a year helps no one. Keep what you query; sample or drop the rest. (Days 35, 83.)

**Honest rule:** telemetry is a budget, not a buffet. Spend it where the questions are. A system that emits a little of the right data beats a system that emits a flood nobody can afford to keep.

---

## 6. Hands-on: emit all three signals from one request (20 minutes)

You need only **Python 3** — nothing to install. You will make one fake checkout request produce a log, a metric, and a trace span, then watch the same `trace_id` tie them together.

**Step 1 — Save this as `telemetry_demo.py` and run it.**

```python
#!/usr/bin/env python3
"""Emit the three kinds of telemetry for ONE fake checkout request."""
import json, time, uuid

trace_id = uuid.uuid4().hex          # 32 hex chars, like a real trace ID
span_id = uuid.uuid4().hex[:16]      # 16 hex chars
now_ns = int(time.time() * 1e9)

# --- 1. A LOG: one line, structured JSON, with the trace ID inside ---
log = {
    "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
    "level": "INFO",
    "service": "checkout",
    "message": "order placed",
    "trace_id": trace_id,
    "user_id": "user-4821",
    "order_total_usd": 120,
    "latency_ms": 342,
}
print("--- LOG ---")
print(json.dumps(log))

# --- 2. A METRIC: Prometheus text format ---
print("\n--- METRIC (Prometheus exposition format) ---")
print("# HELP checkout_orders_total Total completed checkouts")
print("# TYPE checkout_orders_total counter")
print('checkout_orders_total{status="success",region="eu-west"} 1')
print("# HELP checkout_latency_seconds Checkout latency in seconds")
print("# TYPE checkout_latency_seconds histogram")
print('checkout_latency_seconds_bucket{le="0.5"} 1')
print('checkout_latency_seconds_bucket{le="+Inf"} 1')
print('checkout_latency_seconds_sum 0.342')
print('checkout_latency_seconds_count 1')

# --- 3. A TRACE SPAN: OTel-shaped JSON ---
span = {
    "traceId": trace_id,
    "spanId": span_id,
    "name": "POST /checkout",
    "kind": 2,  # 2 = server span (1 = client, 3 = internal)
    "startTimeUnixNano": str(now_ns),
    "endTimeUnixNano": str(now_ns + 342_000_000),
    "attributes": [
        {"key": "http.method", "value": {"stringValue": "POST"}},
        {"key": "http.status_code", "value": {"intValue": "200"}},
        {"key": "service.name", "value": {"stringValue": "checkout"}},
    ],
    "status": {"code": "STATUS_CODE_OK"},
}
print("\n--- TRACE SPAN (OpenTelemetry shape) ---")
print(json.dumps(span, indent=2))
print("\nSame trace_id in the log and the span:", trace_id)
```

```bash
python3 telemetry_demo.py
```

Expected output (your `trace_id` and timestamps will differ — that is normal, they are generated fresh each run):

```
--- LOG ---
{"timestamp": "2026-11-22T13:05:59Z", "level": "INFO", "service": "checkout", "message": "order placed", "trace_id": "64be4d010d8f4fac8c367d31ba28907e", "user_id": "user-4821", "order_total_usd": 120, "latency_ms": 342}

--- METRIC (Prometheus exposition format) ---
# HELP checkout_orders_total Total completed checkouts
# TYPE checkout_orders_total counter
checkout_orders_total{status="success",region="eu-west"} 1
# HELP checkout_latency_seconds Checkout latency in seconds
# TYPE checkout_latency_seconds histogram
checkout_latency_seconds_bucket{le="0.5"} 1
checkout_latency_seconds_bucket{le="+Inf"} 1
checkout_latency_seconds_sum 0.342
checkout_latency_seconds_count 1

--- TRACE SPAN (OpenTelemetry shape) ---
{
  "traceId": "64be4d010d8f4fac8c367d31ba28907e",
  "spanId": "6445626e26dd48f0",
  "name": "POST /checkout",
  "kind": 2,
  "startTimeUnixNano": "1795352759871539968",
  "endTimeUnixNano": "1795352760213539968",
  "attributes": [
    {"key": "http.method", "value": {"stringValue": "POST"}},
    {"key": "http.status_code", "value": {"intValue": "200"}},
    {"key": "service.name", "value": {"stringValue": "checkout"}}
  ],
  "status": {"code": "STATUS_CODE_OK"}
}

Same trace_id in the log and the span: 64be4d010d8f4fac8c367d31ba28907e
```

Three things to notice:
- The **log** is one JSON line — structured, so a machine can search it.
- The **metric** is in Prometheus exposition format — plain text a scraper can read. The `{status="success",region="eu-west"}` part is the context that lets you slice it.
- The **span** is OTel-shaped JSON — and its `traceId` is *the same value* as the log's `trace_id`.

**Step 2 — Follow the thread.** Save the output and pull the trace ID out of it. The same ID should appear in the log, the span, and the last line — that is correlation, done by hand:

```bash
python3 telemetry_demo.py > telemetry.txt
TRACE=$(grep -o '"trace_id": "[a-f0-9]*"' telemetry.txt | head -1 | cut -d'"' -f4)
grep "$TRACE" telemetry.txt
```

Expected output (your ID will differ):

```
{"timestamp": "2026-11-22T13:06:28Z", "level": "INFO", "service": "checkout", "message": "order placed", "trace_id": "4c9d5230a9af4147b605c5678ed08e37", "user_id": "user-4821", "order_total_usd": 120, "latency_ms": 342}
  "traceId": "4c9d5230a9af4147b605c5678ed08e37",
Same trace_id in the log and the span: 4c9d5230a9af4147b605c5678ed08e37
```

This is exactly what a real backend does when you click "view trace" from a log line: it searches for the shared ID. No shared ID, no jump.

**Step 3 — Ask which signal answers which question.**

| Question | Best signal | Why |
|---|---|---|
| "How slow is checkout right now?" | Metric | Numbers over time; one query gives the trend |
| "What exactly happened for order 120?" | Log | Words in order; the full story of that request |
| "Which step inside checkout was slow?" | Trace | The waterfall shows every step and its time |

If one signal cannot answer your question, you probably need a different signal — not a louder version of the same one.

**Step 4 (optional) — Send a real span to a real Collector.** If you have Docker, run the OpenTelemetry Collector with the `debug` exporter — it prints everything it receives to its own log, so you can watch a real pipeline work:

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
exporters:
  debug:
    verbosity: detailed
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      exporters: [debug]
    logs:
      receivers: [otlp]
      exporters: [debug]
```

```bash
docker run --rm \
  -p 127.0.0.1:4317:4317 \
  -p 127.0.0.1:4318:4318 \
  -v "$PWD/otel-collector.yaml:/etc/otelcol-contrib/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.158.0
```

Then, in a second terminal, send one span with `curl` to the OTLP/HTTP endpoint. Get a current time in nanoseconds first (`date +%s%N`), and put it in both timestamp fields:

```bash
# span.json — replace the two timestamps with your own `date +%s%N` output
curl -i http://localhost:4318/v1/traces -X POST \
  -H "Content-Type: application/json" -d @span.json
```

```json
{
  "resourceSpans": [{
    "resource": {
      "attributes": [{ "key": "service.name", "value": { "stringValue": "checkout" } }]
    },
    "scopeSpans": [{
      "scope": { "name": "manual-test" },
      "spans": [{
        "traceId": "71699b6fe85982c7c8995ea3d9c95df2",
        "spanId": "a1b2c3d4e5f60708",
        "name": "POST /checkout",
        "kind": 2,
        "startTimeUnixNano": "1795352760000000000",
        "endTimeUnixNano": "1795352760342000000",
        "status": {}
      }]
    }]
  }]
}
```

What to look for: `curl` should answer `HTTP/1.1 200 OK` with an empty body, and the Collector terminal should print your span back — service name, trace ID, span name, and all. That is the full pipeline from Section 2, running on your laptop: emit (curl) → collect (OTLP receiver) → export (debug). A real backend replaces the debug exporter; the shape of the data does not change.

**Step 5 — Try it on your own system.** Pick one service you own and answer these three questions honestly:
1. Does it emit all three signals — metrics, logs, traces — or only one or two?
2. Do its logs carry a `trace_id`? If not, you cannot jump from a log to a trace.
3. What is one question you asked in your last incident that your telemetry could *not* answer? That gap is your next instrumentation task.

---

## 7. Key terms from today

- **Telemetry** — data a system emits about its own behavior while it runs; the raw material of observability.
- **Signal** — one kind of telemetry: metric, log, trace, event, or profile.
- **Attribute (label / tag)** — a key-value pair attached to telemetry (`region=eu-west`) that lets you slice and filter later.
- **Instrumentation** — the code that makes a system emit telemetry: manual (you write it) or automatic (an agent or SDK does it). (Day 7.)
- **Timestamp** — the exact moment a piece of telemetry happened. Without it, you cannot order events or draw a graph.
- **OTLP** — the OpenTelemetry Protocol: the standard way to send telemetry. gRPC on port 4317, HTTP on port 4318.
- **Collector** — a service that receives telemetry, processes it (filter, batch, sample), and forwards it to backends. (Day 45.)
- **Resource** — what produced the telemetry: which service, which host, which version. It travels with every signal.
- **Sampling** — keeping only some of the telemetry (for example, 1 in 100 traces) to control cost. (Day 41.)
- **Cardinality** — how many distinct values a label can take. High cardinality is powerful and expensive. (Day 10.)

---

## 8. Common mistakes

1. **Emitting data nobody can connect.** Logs without a `trace_id`, metrics without the labels you slice by. Each signal works alone, and the incident still takes hours because you cannot jump between them. Always carry the shared IDs.
2. **Logging everything at DEBUG in production.** A flood of debug lines costs money to store, slows the system, and buries the one error line that matters. Log INFO in production; turn DEBUG on only when you are hunting something.
3. **Putting secrets or personal data in telemetry.** Passwords, API keys, credit card numbers, full email addresses in a log line or a span attribute — once emitted, they are copied to every backend, every dashboard, every export. They are almost impossible to erase. Redact before you emit.

---

## 9. Recap — the 5 things to remember

1. Telemetry is the data your system sends out about itself — metrics, logs, traces, and more — and observability is what you do with it.
2. Every piece of telemetry has three parts: what happened, when it happened, and the context (attributes) that lets you slice it later.
3. The three main signals answer different questions: metrics show trends, logs tell the story, traces show the exact slow step — together they solve incidents in minutes.
4. A shared `trace_id` ties the signals into one story. Without shared IDs, each signal is a clue in a different room.
5. Telemetry is a budget, not a buffet: collect richly where failures cost money, hold back on high-cardinality labels, secrets, and noise nobody queries.

**Tomorrow — Day 4:** The three pillars: metrics, logs, traces (overview) — what each one is for, how they differ, and how they work together.
