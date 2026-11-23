# Day 4 — The Three Pillars: Metrics, Logs, Traces

**In one line:** Metrics tell you *how much and how slow*, logs tell you *what happened*, and traces tell you *where the time went* — together, the three pillars let you answer almost any question about your system.

---

## 1. Explain it like I'm five

Imagine you run a pizza shop, and one evening a customer says: "My pizza took an hour. I am angry."

You have three kinds of records in your shop:

- **A counting board (metrics):** "Today we sold 120 pizzas. Average wait: 25 minutes. 8 orders were late." The board shows *numbers over time*. It tells you something is wrong, but not which pizza or why.
- **A diary (logs):** "6:12 PM — Order #84 started. 6:20 PM — Cheese ran out. 6:31 PM — Oven door stuck. 6:58 PM — Order #84 delivered." The diary tells you *what happened, in order, in words*.
- **A delivery tracker (traces):** One pizza's whole journey, step by step: order taken (2 min) → dough made (8 min) → oven (stuck 25 min) → box (3 min) → delivery (20 min). The tracker shows *where the time went for one single order*.

One record alone is not enough. The board says "orders are slow." The diary says "the oven was stuck." The tracker says "Order #84 lost 25 minutes in the oven." Together, you know the full story — and you know it is the oven, not the delivery driver.

That is the three pillars. **Metrics** are the counting board. **Logs** are the diary. **Traces** are the journey of one request. Every observability tool in this series — Prometheus, Loki, Jaeger, Tempo, Grafana — exists to store and search one (or more) of these three.

---

## 2. The real definition

Each pillar is one *signal* — one type of telemetry (Day 3). Each answers a different question:

| Pillar | The question it answers | What it looks like | Example |
|---|---|---|---|
| **Metrics** | *How much? How slow? How full — right now and over time?* | Numbers with labels, sampled over time | `checkout_latency_seconds: 0.34` at 3:05 PM |
| **Logs** | *What happened, and in what order?* | Timestamped lines of text (best as structured JSON) | `{"event": "order_placed", "order_id": "84", "status": "failed"}` |
| **Traces** | *Where did one single request spend its time?* | A tree of **spans** — one span per step, linked by IDs | Checkout request → 8 s total: payment step took 7.5 s |

A few words to learn now (each gets its own deep dive later in the series):

- **Time series:** a metric's values over time. `checkout_latency` at 3:00, 3:01, 3:02… is one time series. Prometheus stores these (Days 11–13).
- **Label:** a tag on a metric, like `status="failed"` or `region="eu-west"`. Labels let you *slice* the numbers. But too many different label values = high **cardinality** — Day 10.
- **Span:** one step inside a trace. A span has a name ("call payment service"), a start time, a duration, and a **span ID**. Spans nest: the checkout span contains the payment span. (Day 40.)
- **Trace ID:** one ID shared by every span in the same request. It is the thread that ties the whole journey together — and the same ID can be written into your log lines, so logs and traces connect.
- **Structured log:** a log line written as JSON, with named fields (`order_id`, `status`), instead of free text. Machines can search JSON fast; free text they cannot. (Day 27.)

### The three pillars, side by side

| | Metrics | Logs | Traces |
|---|---|---|---|
| **Best question** | "Is it getting worse?" | "What exactly happened?" | "Where did this request get slow?" |
| **Strength** | Cheap, fast, great for alerts and graphs | Rich detail, full story | Exact path of one request across services |
| **Weakness** | No detail about *one* event | Huge volume, hard to search without structure | Expensive if you keep 100% of requests |
| **Kept for** | Weeks to months (numbers are small) | Days to weeks (text is big) | Days to weeks (often sampled) |
| **Main tools** | Prometheus, CloudWatch, Datadog | Loki, Elasticsearch, Cloud Logging | Jaeger, Tempo, Zipkin, X-Ray |

**The key idea:** no pillar replaces the others. A metric tells you *something* is slow. A trace tells you *which step* is slow. A log tells you *why* that step failed. Real debugging uses all three, jumping between them — and the jump works because they share IDs (trace ID, service name, timestamp).

---

## 3. A real example

It is Friday, 6 PM. Your online shop's checkout is slow. Here is how the three pillars work together:

**1. Metrics — the alarm.**
Your dashboard shows checkout latency (p95) jumped from 0.4 s to 8 s at 5:47 PM. Error rate rose from 0.1% to 12%. This is the *smoke alarm*. It tells you *when* it started and *how bad* it is — but not why.

**2. Metrics — slice to narrow it down.**
You slice the same latency by payment provider: Provider A is fine, Provider B is at 8 s. Slice by region: only one region is hit. Now you know *where* to look. Still no root cause.

**3. Traces — follow one slow request.**
You open one slow checkout's trace. It shows: `checkout (8.2 s)` → `validate cart (0.1 s)` → `charge card via Provider B (7.8 s)` → `send confirmation (0.3 s)`. The payment step ate 95% of the time. *Where* the time went: found.

**4. Logs — read the exact failure.**
You search the logs for that trace ID and that minute. One line: `{"event": "payment_retry", "provider": "B", "attempt": 3, "error": "timeout after 2s"}`. The payment service retried 3 times, each timing out after 2 seconds. Root cause: Provider B is timing out; your retry loop multiplies the wait. Fix: shorten the timeout, retry once, fail fast.

Notice the order: **metrics found it → traces located it → logs explained it**. That is the standard incident flow, and it is why you need all three pillars — not just your favorite one.

---

## 4. What problem does it solve?

Each pillar alone leaves a blind spot:

- **Metrics only:** you know checkout is slow, but you cannot see *which* request, *which* step, or *why*. You guess.
- **Logs only:** you have the full story, but finding it means searching millions of lines — and you only know what to search for *after* you already suspect the cause.
- **Traces only:** you can follow one request perfectly, but you cannot see the trend — is this one slow request, or are *all* requests slow? Is it getting worse?

Together, they solve the core problem of Day 2 (unknown-unknowns): **any question about a failure can be answered, because the three views cover each other's blind spots.** Metrics show the shape of the problem over time. Traces show the path of one bad request. Logs show the exact words of the failure.

---

## 5. When to use each — and when NOT to

**Use metrics when:**
- You need an alert ("wake me if error rate > 5%") or a graph over time.
- You need to compare *now* vs *yesterday* vs *last week*.
- The question is about amounts: how many, how slow, how full.

**Do NOT use metrics when:**
- You need to know what happened to *one specific* user or order. Metrics aggregate — they throw away individual detail by design.
- Your label would have thousands of values (user IDs, order IDs, URLs). That is a cardinality bomb — Day 10. Put IDs in logs and traces, not metrics.

**Use logs when:**
- You need the exact story: what happened, in order, with the details.
- Something failed and you need the error message, the input, the context.
- You need an audit trail ("who changed what, when?").

**Do NOT use logs when:**
- You need a graph or an alert on a trend. Counting log lines to build a graph is slow and expensive — emit a metric instead.
- You log every tiny step at DEBUG level in production. That is noise, cost, and disk — and it buries the lines that matter. (Day 35: sampling and cost control.)

**Use traces when:**
- A request crosses services, and you need to know *which step* is slow or failing.
- You need to answer "why was *this one* request slow?" for one real user.

**Do NOT use traces when:**
- The system is one small service on one machine. A log line with a duration is enough — tracing adds cost and moving parts for little gain.
- You try to keep 100% of traces in a high-traffic system. Trace storage is expensive; sample smartly (Day 41).

**Honest rule:** start with metrics (cheapest, most alerts), add structured logs (the story), then add traces when requests cross service boundaries. That is also the order of this series: metrics (Days 11–25), logs (Days 26–38), traces (Days 39–50).

---

## 6. What about events and profiles?

You may hear of two more signals: **events** ("a deploy happened at 5:47 PM" — very useful, because that is exactly when latency jumped) and **profiles** (which function used the CPU — great for "the code is slow, but where?"). They are real and valuable — but they are *extra* signals, not pillars. Day 5 covers them.

---

## 7. Hands-on: one tiny service, all three pillars (30 minutes)

Build a small Python checkout service that emits all three pillars at once. You need only Python 3 and two libraries — no Docker, no server, no account.

**Step 1 — Install the libraries.**

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install prometheus_client opentelemetry-sdk
```

**Step 2 — Save this as `three_pillars.py`.**

```python
# three_pillars.py — one tiny checkout service, all three pillars.
import json, logging, random, time
from prometheus_client import Counter, Histogram, start_http_server
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor

# --- Traces: spans print to the console as JSON ---
provider = TracerProvider(resource=Resource.create({"service.name": "checkout"}))
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

# --- Metrics: served on http://localhost:8000/metrics ---
CHECKOUTS_TOTAL = Counter("checkout_orders_total", "Orders placed", ["status"])
CHECKOUT_SECONDS = Histogram(
    "checkout_duration_seconds", "Time to finish one checkout",
    buckets=(0.05, 0.1, 0.25, 0.5, 1.0, 2.5),
)

# --- Logs: one JSON line per event ---
logging.basicConfig(level=logging.INFO, format="%(message)s")
log = logging.getLogger("checkout")

def place_order(order_id: str) -> None:
    with tracer.start_as_current_span("checkout") as span:
        trace_id = format(span.get_span_context().trace_id, "032x")
        start = time.monotonic()
        time.sleep(random.uniform(0.05, 0.4))  # pretend work: charge the card
        ok = random.random() > 0.2
        status = "success" if ok else "failed"
        CHECKOUTS_TOTAL.labels(status=status).inc()
        CHECKOUT_SECONDS.observe(time.monotonic() - start)
        log.info(json.dumps({
            "level": "INFO", "event": "order_placed",
            "order_id": order_id, "status": status,
            "trace_id": trace_id, "service": "checkout",
        }))

if __name__ == "__main__":
    start_http_server(8000)
    for i in range(5):
        place_order(f"order-{1000 + i}")
        time.sleep(1)
```

**Step 3 — Run it.**

```bash
python three_pillars.py
```

**Step 4 — See each pillar.**

*Metrics* — in a second terminal, while it runs:

```bash
curl -s http://localhost:8000/metrics | grep checkout_
```

Expected output (your numbers will differ):

```text
# HELP checkout_orders_total Orders placed
# TYPE checkout_orders_total counter
checkout_orders_total{status="failed"} 1.0
checkout_orders_total{status="success"} 4.0
# HELP checkout_duration_seconds Time to finish one checkout
# TYPE checkout_duration_seconds histogram
checkout_duration_seconds_bucket{le="0.05"} 0.0
checkout_duration_seconds_bucket{le="0.25"} 3.0
checkout_duration_seconds_bucket{le="0.5"} 5.0
checkout_duration_seconds_bucket{le="+Inf"} 5.0
checkout_duration_seconds_count 5.0
checkout_duration_seconds_sum 1.13
```

Read it: 5 orders, 4 succeeded, 1 failed; 3 finished in under 0.25 s. That is a metric — a number over time, sliced by `status`.

*Logs* — one JSON line per order, in the console:

```text
{"level": "INFO", "event": "order_placed", "order_id": "order-1001", "status": "success", "trace_id": "a3f9c1e2b8474d6a9c2e1f0b8d4a6c12", "service": "checkout"}
```

*Traces* — one JSON span per order, also in the console (trimmed here):

```json
{
    "name": "checkout",
    "context": {
        "trace_id": "0xa3f9c1e2b8474d6a9c2e1f0b8d4a6c12",
        "span_id": "0x7e2b91c04d5a6f83"
    },
    "kind": "SpanKind.INTERNAL",
    "parent_id": null,
    "start_time": "2026-11-23T13:00:01.123456Z",
    "end_time": "2026-11-23T13:00:01.341234Z"
}
```

**Step 5 — See the connection.** The `trace_id` in the log line (`a3f9c1e2...`) matches the `trace_id` in the span (`0xa3f9c1e2...`). Same request, two pillars, one ID. That shared ID is what lets a real tool jump from a log line to its trace, or from a trace to its logs.

**Step 6 — Ask a metric question.** With a Prometheus server scraping `:8000` (Day 13), this query gives the failure rate:

```promql
rate(checkout_orders_total{status="failed"}[5m])
  /
rate(checkout_orders_total[5m])
```

That is the same question as the dashboard in section 3 — "how bad is it?" — answered by numbers, not by reading logs.

**Try it yourself:** change one thing — make the failure chance `random.random() > 0.8` (80% fail) — and watch the metric, the log, and the trace all change together. Same code, three views.

---

## 8. Key terms from today

- **Metric** — a number over time, with labels. Answers "how much / how slow / how full?"
- **Log** — a timestamped record of an event, best written as structured JSON. Answers "what happened?"
- **Trace** — the journey of one request across services, made of spans. Answers "where did the time go?"
- **Span** — one step inside a trace: a named unit of work with a start, a duration, and a span ID.
- **Trace ID** — the ID shared by all spans of one request; also written into logs so the pillars connect.
- **Time series** — one metric's values over time; what Prometheus stores.
- **Label** — a key-value tag on a metric (`status="failed"`) used to slice it. (Cardinality: Day 10.)
- **Structured log** — a log line as JSON with named fields, so machines can search it.
- **Pillar** — one of the three core signals: metrics, logs, traces. (Events and profiles are extra signals — Day 5.)

---

## 9. Common mistakes

1. **Treating one pillar as enough.** "We have logs, so we are observable." Logs without metrics means no alerts and no trends; metrics without logs means no story; both without traces means no path through services. You need all three — they cover each other's blind spots.
2. **Putting high-cardinality data in metrics.** User IDs, order IDs, and full URLs as metric labels explode your series count and your bill (Day 10). IDs belong in logs and traces, where they cost little.
3. **Logs and traces that cannot find each other.** If your log lines do not carry the trace ID, the jump from "this error" to "this request's trace" is manual guesswork. Always log the trace ID (and service name) with every request-scoped log line — it costs one field and saves hours.

---

## 10. Recap — the 5 things to remember

1. Metrics = numbers over time ("how much, how slow"); logs = events in words ("what happened"); traces = one request's journey ("where did the time go").
2. The standard incident flow is metrics find it → traces locate it → logs explain it.
3. The pillars connect through shared IDs: trace ID, service name, timestamp.
4. Each pillar has a job it is bad at: metrics cannot tell one user's story, logs cannot graph trends cheaply, traces are too expensive to keep at 100%.
5. Start with metrics, add structured logs, add traces when requests cross services — in that order.

**Tomorrow — Day 5:** Signals beyond the pillars: events, profiles, and dumps — the extra signals that fill the gaps the big three leave behind.
