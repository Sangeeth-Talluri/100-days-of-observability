# Day 7 — Instrumentation: Manual vs Automatic

**In one line:** Instrumentation is how telemetry gets out of your code — automatic instrumentation watches your frameworks for free, manual instrumentation records the business details only you know, and a good system uses both.

---

## 1. Explain it like I'm five

Imagine a shop with two kinds of watchers.

The first watcher is a set of **security cameras**. Once you mount them, they record everything on their own: who walks in the door, who opens the fridge, who goes to the checkout counter. You never have to tell the cameras what to do. They see the standard stuff, all day, for free.

The second watcher is a **shopkeeper with a notebook**. The cameras cannot see that a customer returned a shirt because the size was wrong, or that the manager approved a $50 discount for a regular customer, or that the card machine failed twice before the payment went through. Only the shopkeeper knows those things — so the shopkeeper writes them down by hand.

Instrumentation works exactly like that.

**Automatic instrumentation** is the cameras: an agent or a library watches your frameworks — your web server, your database driver, your HTTP client — and records a span for every request and every query, with no code changes from you.

**Manual instrumentation** is the notebook: you write a few lines of code to record the things only your business logic knows — which payment provider was used, how much the order was worth, why a retry happened.

A shop with only cameras knows *what happened* but not *why*. A shop with only a notebook misses most of what happens. You need both — and Day 6's OpenTelemetry gives you one standard API for both kinds.

---

## 2. The real definition

**Instrumentation** is the act of adding code (or an agent) to a program so that it produces telemetry: spans, metrics, and logs. There are two ways to do it.

### Automatic instrumentation (also called zero-code instrumentation)

You do not change your source code. Instead, a tool patches your program while it runs:

| Language | How it works | How you turn it on |
|---|---|---|
| **Java** | An agent JAR rewrites your code's bytecode as the JVM loads it | `java -javaagent:opentelemetry-javaagent.jar -jar app.jar` |
| **Python** | A launcher wraps your app and patches libraries at import time | `opentelemetry-instrument python app.py` |
| **Node.js** | A register hook loads instrumentation before your app starts | `node --require @opentelemetry/auto-instrumentations-node/register app.js` |
| **.NET** | An auto-instrumentation package patches the runtime | The `OpenTelemetry.AutoInstrumentation` package and its launcher |

The agent knows the frameworks you use — Spring, Flask, Express, JDBC, the `requests` library, Kafka clients — and creates spans for them. One HTTP request in, one span out. One database query, one span. Context (the trace ID) is passed along automatically, so a trace follows a request across services without you writing a single line.

### Manual instrumentation

You write code with the OpenTelemetry API, in the places where the *meaning* of the work lives:

```python
tracer = trace.get_tracer("checkout.tracer")

with tracer.start_as_current_span("charge-card") as span:
    span.set_attribute("payment.provider", "card-provider-b")
    span.add_event("payment.retry", {"attempt": 2})
```

That is three API ideas, and they cover almost everything manual instrumentation ever does:

- **Start a span** around a unit of work that matters to the business (`charge-card`, `validate-address`, `reserve-seat`).
- **Set attributes** — small facts about that work (`payment.provider`, `order.total`, `feature.flag`). Attributes are how you filter and group later.
- **Add events** — timestamped notes inside a span (`payment.retry`, `cache.miss`, `fallback.used`). Events are the notebook entries inside one operation.

The same API also creates **custom metrics**: a counter for completed orders, a histogram for payment latency, a gauge for the number of items in a queue. Auto-instrumentation gives you runtime metrics (CPU, memory, request counts). It cannot know what an "order" is — only your code knows that.

### How the two fit together

Think of it as a picture frame and a picture:

- **Auto-instrumentation builds the frame.** Every request gets a server span, every database call gets a child span, every outgoing HTTP call gets a child span. The trace has a shape before you write anything.
- **Manual instrumentation paints the picture.** Your business spans hang *inside* the auto spans as children. The trace now says not just "POST /checkout took 2.4 seconds" but "POST /checkout took 2.4 seconds, and 2.1 of it was `charge-card` retrying twice against card-provider-b."

That nesting is the whole trick. A manual span created inside a request automatically becomes a child of the auto span, because the OpenTelemetry API keeps the *current span* in context. You do not wire parent and child together by hand — you just create the span while the request is running, and the SDK does the rest.

---

## 3. A real example

Your team runs a checkout service. One morning, the p99 latency graph jumps from 400 ms to 3 seconds. The on-call engineer opens a slow trace.

**With auto-instrumentation only**, the trace shows:

```text
POST /checkout — 3.1 s
├── SELECT FROM orders — 40 ms
├── POST https://payments.example.com/charge — 2.9 s
└── UPDATE inventory — 60 ms
```

That is useful: the payment call is slow. But the engineer still does not know *why*. Is it all payments, or one provider? Is it retries, or one slow call? Is it first-time charges, or refunds?

**With manual instrumentation added**, the same trace shows:

```text
POST /checkout — 3.1 s
├── SELECT FROM orders — 40 ms
├── charge-card — 2.9 s            ← manual span
│   ├── payment.provider = card-provider-b
│   ├── payment.retry (attempt 1) — 900 ms
│   ├── payment.retry (attempt 2) — 900 ms
│   └── payment.retry (attempt 3) — 900 ms
└── UPDATE inventory — 60 ms
```

Now the answer is on the screen: provider B is timing out at 900 ms, and the code retries three times. The fix is a shorter timeout, or a fallback to provider A — and the engineer found it in one trace instead of an hour of guessing.

That is the division of labor. Auto-instrumentation found *where* the time went. Manual instrumentation explained *why*. Neither one alone would have been enough.

---

## 4. What problem does it solve?

**The blank-trace problem.** A service with no instrumentation produces no traces at all. Auto-instrumentation fixes that in minutes: attach an agent, and every service starts emitting traces the same day — including old services nobody wants to touch, and third-party code you cannot modify.

**The meaningless-trace problem.** A service with only auto-instrumentation produces traces that describe the plumbing, not the business. "POST /checkout took 2 seconds" tells you nothing about orders, payments, or retries. Manual instrumentation adds the business facts that turn a trace from a timeline into an explanation.

**The rewrite problem.** Without a standard, each team invents its own way to record telemetry, and the data cannot be joined across services. Because both kinds of instrumentation speak the same OpenTelemetry API and the same protocol, a trace that starts in a Java service (auto-instrumented) can continue into a Python service (manually instrumented) with one trace ID. Day 6 covered why that standard matters; this is where you see it working.

**The cost problem.** Manual instrumentation is the expensive kind: every span you write is code to maintain, and every span you emit is data to store. Auto-instrumentation is cheap coverage — you get the frame for free, and you spend your manual effort only where the business meaning lives. That is the cheapest way to get traces that are actually useful.

---

## 5. When to use it — and when NOT to

**Use automatic instrumentation when:**
- You want coverage fast — a new service, an old service, a service you did not write. Attach the agent and you have traces today.
- You want the standard stuff: HTTP requests, database queries, queue messages, gRPC calls. The agent already knows these libraries better than you will ever hand-write them.
- You want context propagation for free. The agent passes trace IDs across service boundaries, so traces stay whole without you thinking about headers.

**Use manual instrumentation when:**
- The fact only your code knows: order totals, payment providers, feature flags, tenant IDs, retry reasons, which fallback ran.
- You need a custom metric: orders per minute, payment latency by provider, queue depth, items per cart. Auto-instrumentation cannot invent business metrics.
- You need to mark a decision point inside a span: an event that says "cache missed, fell back to database" or "used the backup provider."

**Do NOT use manual instrumentation when:**
- Auto-instrumentation already covers the operation. Do not wrap an HTTP handler in a manual span when the agent already creates a server span for it — you get two spans saying the same thing, and the trace gets noisy.
- The thing you want to record is a function call, a loop iteration, or a line of code. Spans are for units of work a human would ask about, not for every function. A span per loop iteration is a cost explosion (Day 10, on cardinality, shows the bill).
- You are tempted to log inside a span. Logs belong in the logs pipeline, linked to the trace by the trace ID — not stuffed into span attributes. (Day 36, on correlation, shows the right way.)

**The honest rule:** start with auto-instrumentation everywhere — it is free coverage. Then add manual instrumentation in the five or ten places per service where the business meaning lives. If you find yourself writing manual spans in more places than that, you are probably re-instrumenting what the agent already does.

---

## 6. Hands-on: auto vs manual on the same service (45 minutes)

This builds directly on Day 6. You need the same setup: Docker, Python 3.10+, the Collector running from Day 6 (`otelcol.yaml`, `debug` exporter, port 4317), and the Flask app from Day 6. If the Collector is not running, start it again with the Day 6 command.

### Part A — Auto-instrumentation: watch the framework create spans you never wrote

**Step 1 — Add a second route that calls another service.** Save this as `app.py` (the Day 6 app, plus one new route that calls an outside URL with the `requests` library):

```python
import requests
from flask import Flask

app = Flask(__name__)

@app.route("/checkout", methods=["POST"])
def checkout():
    # Pretend this calls a payment provider.
    resp = requests.get("https://httpbin.org/status/200", timeout=5)
    return {"status": "paid", "order_id": "ord-1234",
            "provider_status": resp.status_code}, 200

@app.route("/health")
def health():
    return {"status": "ok"}, 200

if __name__ == "__main__":
    app.run(port=5000)
```

**Step 2 — Install the app and the OTel distro.**

```bash
pip install flask requests opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

`opentelemetry-bootstrap` scans your installed packages and installs the matching instrumentation libraries — because `flask` and `requests` are installed, it adds the Flask *and* the Requests instrumentation for you. Nothing in your code mentions OpenTelemetry.

**Step 3 — Run the app through the auto-instrumentation launcher.**

```bash
OTEL_SERVICE_NAME=checkout \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
opentelemetry-instrument python app.py
```

**Step 4 — Send a request and read the Collector output.**

```bash
curl -s -X POST http://localhost:5000/checkout
```

In the Collector's terminal you will see **two spans for one request**, and you wrote neither of them:

```text
Span #0
    Name           : POST /checkout
    Kind           : Server
    InstrumentationScope flask
    Attributes:
         -> http.request.method: Str(POST)
         -> http.route: Str(/checkout)
         -> http.response.status_code: Int(200)

Span #1
    Name           : GET
    Kind           : Client
    Parent ID      : <the ID of Span #0>
    InstrumentationScope opentelemetry-instrumentation-requests
    Attributes:
         -> http.request.method: Str(GET)
         -> url.full: Str(https://httpbin.org/status/200)
         -> http.response.status_code: Int(200)
```

Two things to notice:

1. **The client span is a child of the server span.** You did not wire that up — the Requests instrumentation read the current span from context and made its span a child. That is the frame, built for free.
2. **The span names come from the framework, not from you.** `POST /checkout` is a route, `GET` is an outgoing call. Useful, but generic. It does not say "payment," and it does not say which provider. That is the gap manual instrumentation fills.

### Part B — Manual instrumentation: add the business meaning

**Step 1 — Add a manual span and a custom metric to `app.py`.** This is the same pattern from Day 6, Part C — now you can see why it exists:

```python
import requests
import time
from flask import Flask
from opentelemetry import trace, metrics

tracer = trace.get_tracer("checkout.tracer")
meter = metrics.get_meter("checkout.meter")
payment_latency = meter.create_histogram(
    "checkout.payment.duration",
    unit="ms",
    description="Time spent charging the card, by provider",
)

app = Flask(__name__)

@app.route("/checkout", methods=["POST"])
def checkout():
    with tracer.start_as_current_span("charge-card") as span:
        span.set_attribute("payment.provider", "card-provider-b")
        start = time.time()
        resp = requests.get("https://httpbin.org/status/200", timeout=5)
        elapsed_ms = (time.time() - start) * 1000
        span.add_event("payment.succeeded", {"attempt": 1})
        payment_latency.record(elapsed_ms, {"payment.provider": "card-provider-b"})
    return {"status": "paid", "order_id": "ord-1234",
            "provider_status": resp.status_code}, 200
```

Three manual moves, each doing a job the agent cannot:

- `start_as_current_span("charge-card")` — a span named after the *business operation*, not the HTTP method. It nests inside the auto `POST /checkout` span automatically.
- `set_attribute("payment.provider", ...)` — a business fact you can filter on later ("show me all traces where provider is B").
- `payment_latency.record(...)` — a custom metric, split by provider, that no agent could invent because only your code knows what a payment is.

**Step 2 — Restart the app the same way as Part A, Step 3, and send another request.**

In the Collector output you will now see **three spans**, plus a metric:

```text
Span #0  POST /checkout                     (auto — Flask)
Span #1  charge-card                        (manual — yours)
         Attributes: payment.provider = card-provider-b
         Event: payment.succeeded, attempt = 1
Span #2  GET                                (auto — requests, child of charge-card)
```

Notice the nesting: the `GET` client span is now a child of your manual `charge-card` span, because the request happened *inside* it. The frame and the picture fit together — auto spans above and below, your manual span in the middle saying what the work actually was.

And in the metrics pipeline, a histogram called `checkout.payment.duration` with the label `payment.provider=card-provider-b`. That is a metric you can alert on ("payment latency for provider B above 1 second") — and it exists only because you wrote three lines of code.

### Part C — The same idea in Java and Node.js (no code changes)

Auto-instrumentation is a launch-time decision, not a code decision. The pattern is the same in every language:

**Java — attach the agent JAR:**

```bash
curl -L -O https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

OTEL_SERVICE_NAME=checkout \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
java -javaagent:opentelemetry-javaagent.jar -jar app.jar
```

The agent instruments 150+ libraries — Spring, JDBC, Kafka, gRPC — with no dependency changes and no code changes. (Spring Boot 4 also ships its own official starter, `spring-boot-starter-opentelemetry`, if you prefer the SDK route instead of the agent.)

**Node.js — use the register hook:**

```bash
npm install @opentelemetry/auto-instrumentations-node

OTEL_SERVICE_NAME=checkout \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
node --require @opentelemetry/auto-instrumentations-node/register app.js
```

One flag, and HTTP, Express, PostgreSQL, Redis, and dozens of other libraries start producing spans. (For ESM apps, use the loader hook instead — see the official Node.js docs.)

**Go — the honest exception:** Go has no official auto-instrumentation SDK. You instrument Go services manually with the OTel Go API, or you use eBPF-based auto-instrumentation (Beyla, Day 48) which watches the program from outside, without touching its code.

### Part D — Controlling auto-instrumentation (turn pieces off)

Auto-instrumentation is not all-or-nothing. When one library produces noise — or a library you do not want traced — you switch it off with configuration, not code:

```bash
# Python: disable one instrumentation by name
OTEL_PYTHON_DISABLED_INSTRUMENTATIONS=flask \
opentelemetry-instrument python app.py

# Node.js: disable one instrumentation by name
OTEL_NODE_DISABLED_INSTRUMENTATIONS=http \
node --require @opentelemetry/auto-instrumentations-node/register app.js

# Java: disable one instrumentation with a system property
java -Dotel.instrumentation.jdbc.enabled=false \
     -javaagent:opentelemetry-javaagent.jar -jar app.jar
```

And when the *volume* is the problem rather than one library, sampling throttles how much you keep — this is configuration, not code, and it applies to both auto and manual spans:

```bash
# Keep roughly 10% of traces, decided by trace ID so whole traces stay whole
OTEL_TRACES_SAMPLER=parentbased_traceidratio \
OTEL_TRACES_SAMPLER_ARG=0.1 \
opentelemetry-instrument python app.py
```

`parentbased` means: if a parent service already decided to keep a trace, the child keeps it too — so sampled traces stay complete instead of arriving in pieces. Day 41 goes deep on sampling strategies.

**Try it yourself:** run the Part B app with `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS=requests`. Send a request. The `GET` client span disappears, but your manual `charge-card` span stays — proof that the two kinds of instrumentation are independent layers, and you control each one separately.

---

## 7. Key terms from today

- **Instrumentation** — adding code or an agent to a program so it produces telemetry (spans, metrics, logs).
- **Automatic instrumentation (zero-code instrumentation)** — telemetry produced by an agent or library that patches your frameworks at runtime, with no changes to your source code. Examples: the Java agent, Python `opentelemetry-instrument`, Node's register hook.
- **Manual instrumentation** — spans, attributes, events, and metrics you create yourself in code, using the OpenTelemetry API, for your business logic.
- **Instrumentation library** — a small package that knows how to instrument one framework or client (Flask, JDBC, `requests`, Kafka). Auto-instrumentation is a bundle of these.
- **Bytecode instrumentation** — how the Java agent works: it rewrites a program's compiled code as the JVM loads it, inserting span-creation calls around library methods.
- **Tracer / Meter** — the API objects you ask for in code (`trace.get_tracer(...)`, `metrics.get_meter(...)`) to create spans and metrics manually.
- **Attribute** — a key-value fact attached to a span or metric (`payment.provider`, `http.route`). Low-cardinality facts only — never user IDs or raw URLs (Day 10).
- **Event** — a timestamped note inside a span (`payment.retry`, `cache.miss`). The notebook entry for one moment inside an operation.
- **Current span / context** — the span that is active right now in this request. A manual span created while a request runs automatically becomes a child of the current (usually auto-created) span.
- **Sampling** — keeping only a fraction of traces (for example, 10%) to control volume and cost, while keeping whole traces whole.
- **eBPF instrumentation** — observing a program from outside the kernel, with no agent inside it and no code changes (Beyla, Day 48). The third way, after auto and manual.

---

## 8. Common mistakes

1. **Wrapping what the agent already spans.** Do not create a manual span around an HTTP handler that already has an auto server span. Instead, add attributes to the span that is already there: `trace.get_current_span().set_attribute("tenant.id", ...)`. One span per operation, not two.
2. **Creating spans with no parent.** If you start a manual span in a background thread or a new goroutine without passing the context along, it starts a brand-new trace instead of joining the request's trace. The trace breaks in two, and the incident trail goes cold. Always propagate context — or create the span inside the request's flow.
3. **A span for every function.** Manual spans are for units of work a human would ask about (`charge-card`, `reserve-seat`). A span per function call or per loop iteration floods your backend with useless spans and a bill to match. If you cannot explain why someone would search for the span, do not create it.
4. **High-cardinality or personal data in manual attributes.** A user ID, an email address, or an order ID as an attribute creates a new unique value per request — storage and cost explode, and personal data leaks into your telemetry. Attributes are for low-cardinality facts (`payment.provider`, `http.route`). (Day 10 goes deep.)
5. **Auto-instrumentation only, forever.** The agent gets you coverage in a day, and then teams stop there. Months later, an incident hits and the traces say "POST /checkout took 2 seconds" with no business facts at all. Schedule the manual pass — the five or ten spans per service where the meaning lives — in the same sprint as the agent rollout, not "later."

---

## 9. Recap — the 5 things to remember

1. Instrumentation is how telemetry gets out of your code. Automatic instrumentation is the agent watching your frameworks for free; manual instrumentation is the code you write for the business facts only you know.
2. Auto-instrumentation builds the frame — a server span per request, a client span per outgoing call, context passed along for free. Start here, everywhere, on day one.
3. Manual instrumentation paints the picture — a span named after the business operation, attributes like `payment.provider`, events like `payment.retry`, and custom metrics like payment latency by provider.
4. The two nest: a manual span created inside a request automatically becomes a child of the auto span. Do not wrap what the agent already spans — add attributes to the span that is already there.
5. Control each layer separately: disable noisy libraries with config (`OTEL_PYTHON_DISABLED_INSTRUMENTATIONS`, `OTEL_NODE_DISABLED_INSTRUMENTATIONS`, `-Dotel.instrumentation.<name>.enabled=false`), and control volume with sampling (`OTEL_TRACES_SAMPLER=parentbased_traceidratio`). Go has no official auto-instrumentation SDK — it is manual, or eBPF with Beyla (Day 48).

**Tomorrow — Day 8:** The terms map: observability vs monitoring vs APM vs logging — four words people use as if they mean the same thing, and the real line between each pair.

---

*Sources checked for this lesson: OpenTelemetry Java agent docs (javaagent JAR download and `-javaagent` flag, opentelemetry-java-instrumentation releases); Node.js auto-instrumentation docs (`@opentelemetry/auto-instrumentations-node`, `--require .../register` hook, `OTEL_NODE_DISABLED_INSTRUMENTATIONS`); Python auto-instrumentation (`opentelemetry-distro`, `opentelemetry-bootstrap`, `opentelemetry-instrument`, `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS`); OpenTelemetry SDK environment-variable configuration (`OTEL_TRACES_SAMPLER`, `OTEL_TRACES_SAMPLER_ARG`, `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`); semantic conventions for HTTP attributes. Versions were current as of November 2026 — pin exact versions and re-check the changelog before upgrading.*
