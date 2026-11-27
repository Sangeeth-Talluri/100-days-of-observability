# Day 8 — The Terms Map: Observability vs Monitoring vs APM vs Logging

**In one line:** Four words that sound the same but mean four different jobs — monitoring watches for known problems, logging writes down what happened, APM watches your application code, and observability is the whole ability to ask *any* question about your system.

---

## 1. Explain it like I'm five

Imagine you run a big hospital.

- **Monitoring** is the nurse who checks every patient's temperature and heart rate every hour, and rings a bell if a number goes out of range. She only checks what the doctors told her to check.
- **Logging** is the notebook at every bed. Every nurse writes down what happened: "2 PM — gave medicine. 3 PM — patient coughed." The notebook remembers everything, in order.
- **APM** is the heart monitor stuck to one patient. It watches *that one patient's* body in deep detail — every heartbeat, every breath — and shows it live on a screen.
- **Observability** is the whole hospital working so well that when *any* doctor walks in and asks *any* question — "Which patients got worse after lunch? Show me everything that happened to bed 12 yesterday" — the answer is already there in the notebooks, the monitors, and the charts.

One hospital. Four different jobs. You need all of them, and they are not the same thing.

---

## 2. The real definitions

These four words get mixed up in meetings, in sales calls, and in job ads. Here is the clean map — print it in your head.

### Monitoring

> **Monitoring is watching a short list of things you already decided matter, and raising an alarm when one of them goes wrong.**

- You pick the checks in advance: CPU above 80%, error rate above 1%, disk 90% full.
- It answers: *"Is something broken right now?"*
- It is **pre-defined**: you can only catch problems you thought of before.

### Logging

> **Logging is your system writing down events as they happen, in lines of text, so there is a record to read later.**

- One line per event: a request arrived, a payment failed, a user logged in.
- It answers: *"What exactly happened, and in what order?"*
- Logs are **one signal** — a raw material. Logging alone is not observability, the same way a pile of bricks alone is not a house.

### APM — Application Performance Monitoring

> **APM is tooling that watches your application code from the inside — how slow each function is, which database calls are slow, and how one request travels through your services.**

- It usually works by **instrumenting your code** (automatically or manually — Day 7) and collecting traces and code-level metrics.
- It answers: *"Which part of my code is slow, and where does a request spend its time?"*
- Typical APM tools: Datadog APM, New Relic, Dynatrace, Elastic APM.
- APM is **focused on the application**. It usually cares less about the servers, the network, or the infrastructure underneath.

### Observability

> **Observability is the property of a system that lets you ask *any* question about its behavior — including questions you never planned — and get an answer from the telemetry it already produced.**

- It is the **whole picture**: monitoring + logs + traces + metrics + events, all connected by IDs, all queryable.
- It answers: *"Why is this happening, for whom, and where exactly?"*
- It is **not a product**. It is a property of your system — and of how your team works.

### The map, in one table

| Term | What it does | Question it answers | Scope |
|---|---|---|---|
| Monitoring | Watches pre-chosen numbers, alerts on thresholds | "Is it broken?" | Known problems |
| Logging | Records events as text lines | "What happened, in what order?" | One signal (raw material) |
| APM | Watches application code and request paths | "Which code is slow?" | The application layer |
| Observability | Connects all signals so any question can be answered | "Why, where, and for whom?" | The whole system |

And the relationships:

- **Logging is a part of observability**, not a rival to it. Logs are one of the signals (Day 4).
- **Monitoring is a part of observability** — the part that watches the known problems and wakes you up (Day 1).
- **APM is a tool category** that *helps* observability, mainly with traces and code-level data. It is not the same as observability, because it usually stops at the application and does not cover infrastructure, or ad-hoc questions outside its dashboards.
- **Observability is the goal. The other three are pieces that build it.**

---

## 3. A real example

Your team runs an online shop with 12 services. Checkout is slow for some users, but only sometimes. Four people look at the problem — each with one tool:

- **The monitoring person** opens the dashboard. "Error rate is normal. CPU is fine. Nothing is red." Monitoring did its job — but it can only see what was pre-defined. No alarm was built for "checkout is slow only for users paying with Provider B."
- **The logging person** searches the logs for "checkout" and "timeout" around 3 PM. She finds lines like `payment retry 3 of 3, timeout after 8s`. Now she knows *what* happened — the payment step timed out. But the logs do not show *why* it timed out, or whether it was every user or just some.
- **The APM person** opens the APM tool. It shows the checkout trace: the request spent 8 seconds inside the payment service, in one database query — `SELECT ... WHERE order_id = ?` — that scanned a table with no index. APM shows the *code-level* cause.
- **The observability view** puts it together: the same slow requests, sliced by payment provider, show only Provider B is slow; the trace shows the exact slow query; the logs show the retries; the metric shows the error rate rising only for that provider. One picture, one answer, in minutes.

No single piece solved it. Monitoring said "nothing is red." Logs said "a timeout happened." APM found the slow query. Observability is what happens when all three are connected and you can ask the question that crosses them.

---

## 4. What problem does it solve?

The problem is **confusion about what to buy, what to build, and what to ask for.**

- Teams buy an APM tool and say "we are observable now" — then during the next incident they cannot answer a question outside the APM's dashboards, because the logs and the infrastructure metrics live somewhere else, unconnected.
- Teams collect logs for years and say "we have observability" — then nobody can query them, nothing is connected by request ID, and finding one failed request takes an hour of `grep`.
- Teams build dashboards for everything and say "we monitor everything" — then a brand-new failure arrives that no dashboard covers, and they are back to guessing.

The terms map solves this by giving each word one clear job. When you know the job, you know what is missing:

- No alarms for known problems → you need **monitoring**.
- No record of what happened → you need **logging**.
- No view into slow code → you need **APM** (or tracing — Day 39).
- Cannot answer new questions → you need **observability** — which means connecting the other three, not buying a fourth tool.

---

## 5. When to use each — and when NOT to

**Use monitoring when:**
- The problem is known and you can write a rule for it: disk full, service down, error rate over 1%.
- You need someone woken up at 3 AM. Monitoring is the alarm.

**Do NOT rely on monitoring alone when:**
- The failure is new. Monitoring cannot catch what nobody predicted. That is Day 2's unknown-unknowns problem.

**Use logging when:**
- You need a step-by-step record: what happened, in what order, for one request or one user.
- You need evidence after the fact: audits, debugging, "show me the last 100 lines before the crash."

**Do NOT use logging as your only tool when:**
- You need to see a trend over time ("are errors rising?") — that is a metric's job, not a log's job.
- Your logs have no request ID connecting them. Unconnected logs are a haystack (Day 1, mistake 2).

**Use APM when:**
- Your application is slow and you need to know *which function, which query, which service call* is slow.
- You run microservices and need to see one request's path through them.

**Do NOT expect APM to do everything when:**
- The problem is in the infrastructure (a full disk, a bad network route, a saturated queue) — APM often does not see it.
- You need to ask a question APM never built a view for. APM answers its own questions well; observability answers *yours*.

**Use "observability" as your goal when:**
- You want the whole system — app, infrastructure, logs, metrics, traces — connected and queryable, so any question gets an answer.

**Do NOT treat observability as a product to buy:**
- No vendor sells it in a box. You build it by connecting signals, adding IDs, and keeping the data queryable.

---

## 6. Hands-on: map your own stack (30 minutes)

The goal: draw *your* terms map. Take the tools your team already runs and sort each one into its real job. Most teams find they have three of the four — and the missing piece is the one that hurts them in incidents.

### Step 1 — List what you have

Make a simple file, `terms-map.md`, in your notes or your team wiki:

```markdown
# Our terms map

## Monitoring (watches known problems, alerts)
- Prometheus — alerts on error rate, latency, saturation
- Grafana Alerting — pages on-call

## Logging (records events)
- Loki — all application logs, via Promtail
- CloudWatch Logs — Lambda logs

## APM (watches application code)
- Grafana Tempo — traces from the OpenTelemetry SDK
- (none yet — we have no code-level profiler)

## Observability gaps (questions we still cannot answer)
- Cannot connect a slow request to its logs — no trace ID in log lines yet
- No metrics for the message queue — queue depth is invisible
```

Be honest in the gaps section. That list is your real roadmap.

### Step 2 — Check that each piece is actually alive

A tool that is installed but not running does not count. Run these health checks against your own stack (change the host and port to match yours):

```bash
# Prometheus — monitoring: should answer "Prometheus Server is Healthy."
curl -s http://localhost:9090/-/healthy

# Grafana — dashboards and alerting:
# should answer {"database":"ok", ...}
curl -s http://localhost:3000/api/health

# Loki — logging: should answer "ready"
curl -s http://localhost:3100/ready

# Tempo — traces (APM side): should answer "ready"
curl -s http://localhost:3200/ready
```

What to look for: every check should answer with a healthy response. If one does not answer, that piece of your map is down — and during an incident it will be down exactly when you need it.

### Step 3 — Ask one question per term

Pick a real recent incident (or a made-up one) and try to answer it four ways:

1. **Monitoring:** "Which alert fired, and what threshold did it cross?"
2. **Logging:** "Show me the log lines for one failed request, in order."
3. **APM:** "Which service call or database query was slowest inside that request?"
4. **Observability:** "Was it all users or only some? Only one region? Only one payment provider?"

Write down which of the four questions you *could not* answer. That is your next project — and this series covers each one: logs in Days 26–38, traces in Days 39–50, dashboards in Days 51–60.

---

## 7. Key terms from today

- **Monitoring** — watching pre-chosen signals and alerting when they cross a threshold. Answers "is it broken?"
- **Logging** — writing down events as text lines, in order, so there is a record. One signal of observability.
- **APM (Application Performance Monitoring)** — tooling that watches application code: slow functions, slow queries, request paths. Usually built on traces.
- **Observability** — the property of a system that lets you ask any question about its behavior from the telemetry it already produced. The whole, not a part.
- **Signal** — one type of telemetry: a metric, a log, a trace, an event. (Day 3, Day 4.)
- **Instrumentation** — making code emit telemetry. APM often does this automatically. (Day 7.)

---

## 8. Common mistakes

1. **"We bought an APM tool, so we are observable."** APM is one lens — the application lens. If your logs, infrastructure metrics, and traces are not connected to it, you still cannot answer questions that cross those lines.
2. **Calling everything "monitoring."** When the team uses one word for four jobs, nobody knows what is missing. "We need better monitoring" could mean alarms, logs, traces, or all three — and the fix is different each time. Use the right word; it tells you the right fix.
3. **Collecting logs but never connecting them.** Logs without a request ID or trace ID are just text. The moment you add the ID, the same logs become evidence you can follow from a metric spike to one exact request. (Correlation gets its own deep dive on Day 36.)

---

## 9. Recap — the 5 things to remember

1. Monitoring watches known problems and raises alarms — it answers "is it broken?"
2. Logging records what happened, line by line — it is one signal, not the whole system.
3. APM watches your application code — slow functions, slow queries, request paths.
4. Observability is the whole: all signals, connected, so *any* question gets an answer.
5. When a tool or a vendor promises "observability," check the map: which of the four jobs does it actually do — and which ones are still missing?

**Tomorrow — Day 9:** The observability lifecycle: collect → ship → store → query → visualize → alert, and how telemetry travels from your code to your screen.
