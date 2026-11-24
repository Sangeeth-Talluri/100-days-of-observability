# Day 5 — Signals Beyond the Pillars: Events, Profiles, and Dumps

**In one line:** Metrics, logs, and traces answer *what is wrong* — but events tell you *what changed*, profiles tell you *which line of code is slow*, and dumps freeze a dying process so you can examine it later.

---

## 1. Explain it like I'm five

Imagine a hospital. A sick patient is your service, and the doctors are you, trying to find out what is wrong.

- The **three pillars** (Day 4) are the nurse's regular checks: the temperature chart (**metrics**), the nurse's written notes (**logs**), and following one patient through every department (**traces**).
- An **event** is a note on the chart that says *"3:00 PM — patient got a new medicine."* The temperature was fine at 2:59 and bad at 3:01. The event tells you *what changed* at the exact moment things went wrong.
- A **profile** is an X-ray. The chart says the patient is in pain, but the X-ray shows *exactly which bone is broken* — which part of the body (which function in your code) is working the hardest.
- A **dump** is freezing the patient in a photo at the moment they collapse. You cannot ask a dead process questions — but a heap dump or a thread dump is a snapshot of everything it held in memory at that moment, so you can look at it later, calmly.

The pillars tell you something is wrong. These three extra signals tell you **what changed, where inside the code, and what the process was holding when it died.**

---

## 2. The real definition

Each extra signal answers a question the three pillars cannot:

| Signal | The question it answers | What it looks like | Example |
|---|---|---|---|
| **Event** | *What changed, and when?* | A short, timestamped fact: "deploy v1.42 started at 5:47 PM" | A deploy, a config change, a scale-up, a pod restart, a feature flag flip |
| **Profile** | *Which code is using the CPU / memory — and how much?* | A sampled stack trace: "function X used 62% of CPU" | A flame graph showing `parseJSON` eating most of the CPU |
| **Dump** | *What was the process holding when it crashed or froze?* | A frozen copy of memory or threads at one moment | A heap dump showing a cache that grew to 4 GB; a thread dump showing 200 threads stuck waiting |

### Events — the change log of your system

An event is a **discrete fact with a timestamp**. It is not a measurement over time (metric), not a story (log), and not a journey (trace). It is one line in the history of *changes*:

- **Deploy events:** "v1.42 rolled out to checkout at 5:47 PM."
- **Config changes:** "timeout raised from 2 s to 10 s."
- **Infrastructure events:** "Kubernetes killed pod `checkout-7d9f` — out of memory."
- **Business events:** "Black Friday sale started."
- **Span events:** a small event *inside* a trace — "cache miss for key `user:123`" at one exact point in one request. (OpenTelemetry spans can carry events; see the hands-on part.)

Events are powerful because **most incidents are caused by a change**. When a graph jumps, the first question is always: "What changed at that minute?" An event marker on the graph answers it in one look.

Where events live: Kubernetes keeps its own event stream (`kubectl get events`), and Grafana lets you draw events as vertical lines on any dashboard (**annotations**). Good teams also send deploy events from their CI/CD pipeline automatically, so every graph carries its own history of changes.

### Profiles — the X-ray of your code

A profile is a **sampled snapshot of what your code is doing**. Every few milliseconds, the profiler asks: "Which function is running right now?" After 30 seconds, it counts the answers. The function that appears most often is where your CPU goes.

- **CPU profile:** which functions burn CPU.
- **Memory (heap) profile:** which functions allocate memory, and what is still held.
- **Goroutine / thread profile:** how many threads exist, and what each one is waiting on.
- **Lock profile:** which locks threads fight over.

The classic way to read a profile is a **flame graph**: a picture where each box is a function, the width of the box is how much CPU it used, and boxes stack on top of each other to show who called whom. Wide boxes at the top are your suspects.

The modern way to collect profiles is **continuous profiling**: a small agent samples your services all the time (cheaply — usually 1% or less of CPU) and keeps the profiles, so you can compare *today* against *yesterday*. Tools: **Grafana Pyroscope**, **Parca**, and Go's built-in **pprof**. (Day 78 goes deep on continuous profiling.)

### Dumps — the frozen crime scene

A dump is a **full copy of a process's state at one moment**, written to a file:

- **Heap dump:** every object in memory — what it is, how big it is, and what holds it. The tool for finding memory leaks: if the heap grows forever, the dump shows *which* objects never get freed.
- **Thread dump (goroutine dump):** every thread and its stack — what each thread was doing, and what it was waiting for. The tool for freezes and deadlocks: if 200 threads are all stuck on the same lock, the dump shows it.
- **Core dump:** the whole process memory, for post-mortem debugging with a debugger.

Dumps are heavy and one-off. You do **not** take them all the time — you take one when something is already wrong, or right before you restart a sick process. The golden rule: **take the dump before you restart.** Restarting destroys the evidence.

---

## 3. A real example

It is Friday, 6 PM. Checkout latency jumped from 0.4 s to 8 s at 5:47 PM. You already know the three pillars from Day 4 — here is what the extra signals add:

**1. Events — what changed at 5:47 PM?**
You look at the latency graph. There is a vertical line on it: *"Deploy: checkout v1.42, 5:47 PM."* The spike starts at the exact same minute. That is not a coincidence — it is a suspect. (Without the event marker, you would spend an hour guessing.)

**2. Profiles — which code got slow?**
You open the CPU profile for `checkout` from 5:50 PM and compare it with the one from 5:30 PM. Before the deploy, the widest box was `chargeCard`. After the deploy, a new wide box appears: `parseDiscountRules` — 62% of CPU. The new version added a discount rule that re-parses a big JSON file on *every* request.

**3. Dumps — what is the process holding?**
While you investigate, memory on the checkout pods climbs steadily. You take a heap dump before restarting the worst pod. It shows one cache object holding 4 GB: the discount rules are cached *per request* instead of once. A thread dump shows 200 threads all waiting on the same lock inside that cache.

**The full story:** the deploy at 5:47 PM added a discount feature that parses a big file on every request (profile), caches the result per request so memory grows without limit (heap dump), and locks while doing it (thread dump). Fix: parse once, cache once, no lock on the hot path.

Notice the order: **event found the change → profile found the function → dump found the leak.** Each signal answered a question the pillars alone could not.

---

## 4. What problem does it solve?

The three pillars leave three gaps:

- **"It was fine until 5:47 PM — what happened then?"** Metrics show the jump, but not the cause. Events close that gap: they are the change history of your system.
- **"The service is slow, but *which line of code*?"** A trace shows the slow *step* (the payment call took 7 s). A profile shows the slow *function inside that step* (`parseDiscountRules` took 62% of CPU). Traces stop at the service boundary; profiles go inside the code.
- **"The process died / froze — what was it holding?"** A dead process cannot answer questions. A dump is the frozen evidence you examine after the fact — the only way to diagnose a crash or a leak that you cannot reproduce.

Together, these signals turn "something is wrong" into "this change, this function, this object."

---

## 5. When to use each — and when NOT to

**Use events when:**
- You need to answer "what changed at this minute?" — deploys, config changes, restarts, flag flips.
- You want every graph to carry its own change history. Send deploy events from CI/CD automatically; never rely on memory.

**Do NOT use events when:**
- You need numbers over time or a full story — that is metrics and logs. An event is one fact, not a trend.
- You flood them: an event for every pod heartbeat is noise. Events are for *changes*, not for heartbeats.

**Use profiles when:**
- A service is slow or uses too much CPU/memory, and you need to know *which function* is responsible.
- You want to compare today against yesterday ("did the deploy make this function hotter?").

**Do NOT use profiles when:**
- You profile with a heavy tool that slows production down. Sampling profilers cost ~1% of CPU — if your tool costs more, use a lighter one or sample less often.
- You try to debug a *single* bad request with a profile. Profiles are averages over many samples; one request's story is a trace.

**Use dumps when:**
- A process crashed, leaked memory, or froze, and you need the frozen state to find out why.
- You take the dump *before* restarting the process — after a restart, the evidence is gone.

**Do NOT use dumps when:**
- You take them on a schedule "just in case." A heap dump of a big service can be gigabytes, and writing it can pause the process for seconds. Dumps are a scalpel, not a monitor.
- You store dumps with customer data in them without care — a heap dump can contain passwords, tokens, and personal data. Treat it like a secret, and delete it when you are done.

**Honest rule:** events are cheap — add them first, from day one. Profiles are cheap too once the agent is installed — add them second. Dumps are expensive and manual — keep them as the emergency tool they are.

---

## 6. Hands-on: events, profiles, and dumps (30 minutes)

Three short exercises, one per signal. You need Go installed (`go version` to check) and, for the Kubernetes part, a cluster with `kubectl`.

### Part A — Events: read Kubernetes events, and mark a deploy on a Grafana graph

**Step 1 — See what changed in your cluster.**

```bash
kubectl get events --sort-by=.lastTimestamp -n default
```

Expected output (yours will differ):

```text
LAST SEEN   TYPE      REASON             OBJECT                  MESSAGE
2m          Normal    Scheduled          pod/checkout-7d9f      Successfully assigned default/checkout-7d9f to node-2
2m          Normal    Pulled             pod/checkout-7d9f      Container image "checkout:v1.42" already present
2m          Normal    Created            pod/checkout-7d9f      Created container checkout
1m          Warning   BackOff            pod/checkout-7d9f      Back-off restarting failed container checkout
```

Read it like a change log: a new pod was scheduled, its image pulled, its container created — and then it started crashing. Kubernetes events are free, and they are the first thing to check when a pod misbehaves.

Filter to the interesting ones — warnings only, last hour:

```bash
kubectl get events --sort-by=.lastTimestamp --field-selector type=Warning -n default
```

**Step 2 — Put a deploy marker on a Grafana graph.**

A deploy is only useful as an event if it lands on your graphs. Grafana's annotation API creates one with a single `curl` (replace `GRAFANA_URL` and `GRAFANA_TOKEN` with your own — a service account token with `annotations:create` permission):

```bash
curl -s -X POST "$GRAFANA_URL/api/annotations" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "time": '$(date +%s000)',
    "tags": ["deploy", "checkout"],
    "text": "Deployed checkout v1.42"
  }'
```

Expected response (an ID means it worked):

```json
{"message": "Annotation added", "id": 12}
```

Now the deploy shows as a vertical line on every dashboard that has the Grafana annotations data source. The `time` field is the current time in epoch milliseconds (`date +%s000` gives that). Put this `curl` in your deploy script, and every graph carries its own change history forever.

**Step 3 — Events inside a trace (OpenTelemetry span events).**

Some events are too small for a graph but perfect inside one request. In Python, a span can carry events (this builds on the Day 4 code):

```python
with tracer.start_as_current_span("checkout") as span:
    # ... do the work ...
    span.add_event("cache_miss", {"key": "user:1234"})
    # ... keep working ...
    span.add_event("payment_retry", {"attempt": 2, "provider": "B"})
```

Each event is timestamped inside the span. In a trace viewer you see them as markers on the span's timeline — "the cache missed *here*, then the payment retried *here*." Same idea as a deploy event, at the scale of one request.

### Part B — Profiles: find the hot function in a Go service with pprof

**Step 1 — Save this as `slow_service.go`.**

```go
// slow_service.go — a tiny service with one deliberately slow function.
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof" // registers /debug/pprof/ on the default mux
	"time"
)

func parseDiscountRules() {
	// Pretend work: burns CPU, like a JSON file parsed on every request.
	start := time.Now()
	for time.Since(start) < 80*time.Millisecond {
		_ = fmt.Sprintf("%d", 42)
	}
}

func workHandler(w http.ResponseWriter, r *http.Request) {
	parseDiscountRules() // the suspect
	fmt.Fprintln(w, "ok")
}

func main() {
	// Profiler on localhost only — never expose /debug/pprof/ to the internet.
	go func() { _ = http.ListenAndServe("127.0.0.1:6060", nil) }()
	http.HandleFunc("/work", workHandler)
	_ = http.ListenAndServe(":8080", nil)
}
```

**Step 2 — Run it and give it traffic.**

```bash
go run slow_service.go &
# in a second terminal, hit it for ~30 seconds:
for i in $(seq 1 300); do curl -s http://localhost:8080/work > /dev/null; sleep 0.1; done
```

**Step 3 — Take a 10-second CPU profile and read the top functions.**

```bash
go tool pprof -top http://127.0.0.1:6060/debug/pprof/profile?seconds=10
```

Expected output (your numbers will differ, but the shape will match):

```text
Type: cpu
Time: Nov 24, 2026 at 8:10am (EST)
Duration: 10s, Total samples = 7.9s (79.0%)
Showing nodes accounting for 7.5s, 94.94% of 7.9s total
      flat  flat%   sum%        cum   cum%
     4.90s 62.03% 62.03%      4.90s 62.03%  main.parseDiscountRules
     1.80s 22.78% 84.81%      1.80s 22.78%  runtime.mallocgc
     0.80s 10.13% 94.94%      0.80s 10.13%  fmt.Sprintf
```

Read it: `parseDiscountRules` burned 62% of the CPU — the exact suspect, named, with a number. That is what a profile does: it turns "the service is slow" into "this function is slow."

**Step 4 — See it as a flame graph.**

```bash
go tool pprof -http=:9000 http://127.0.0.1:6060/debug/pprof/profile?seconds=10
```

Open `http://localhost:9000` in your browser. The widest boxes are the hottest functions — `parseDiscountRules` will be the wide bar at the top. Click a box to zoom into it.

**Step 5 — Clean up.**

```bash
kill %1   # stops the go run process
```

### Part C — Dumps: freeze the process and read its threads and heap

Use the same running service from Part B (restart it if you stopped it).

**Step 1 — A goroutine dump: every thread, frozen in text.**

```bash
curl -s http://127.0.0.1:6060/debug/pprof/goroutine?debug=2 | head -40
```

Expected output (trimmed):

```text
goroutine 6 [running]:
main.parseDiscountRules()
	/path/to/slow_service.go:14 +0x3c
main.workHandler(...)
	/path/to/slow_service.go:21 +0x2a
net/http.(*conn).serve(...)
```

Read it: each goroutine's full stack, frozen at one moment. In a real freeze, you would look for many goroutines stuck at the *same* line — for example 200 goroutines all waiting on one lock. That is the deadlock, written out for you.

**Step 2 — A heap dump, and how to find a leak with two of them.**

Take one heap snapshot now, and one a minute later:

```bash
curl -s -o heap1.prof http://127.0.0.1:6060/debug/pprof/heap
# ... wait a minute, under traffic ...
curl -s -o heap2.prof http://127.0.0.1:6060/debug/pprof/heap
```

Now compare them — the diff shows what *grew*:

```bash
go tool pprof -base heap1.prof heap2.prof
```

Inside the pprof prompt, run:

```text
(pprof) top
```

If one function keeps climbing between the two dumps, that function is allocating memory that never gets freed — that is your leak, and the diff is the proof. (One heap dump alone tells you what is held; *two* dumps tell you what is *growing*.)

**The Java equivalent, for reference** — same idea, different commands:

```bash
jstack <pid> > threads.txt                                  # thread dump
jmap -dump:live,format=b,file=heap.hprof <pid>              # heap dump
```

**Try it yourself:** change `parseDiscountRules` so it allocates a big slice on every call and never frees it (`leaked = append(leaked, make([]byte, 1024*1024))` in a package-level slice), take two heap dumps a minute apart, and watch the diff point straight at it.

---

## 7. Key terms from today

- **Event** — a discrete, timestamped fact that something changed: a deploy, a config change, a restart. Answers "what changed, and when?"
- **Annotation** — a Grafana event marker drawn as a vertical line on a graph, so a deploy and a latency spike can be seen together.
- **Span event** — a small timestamped note inside one trace span ("cache miss here"), at the scale of one request.
- **Profile** — a sampled snapshot of what code is doing: which functions use CPU or memory, and how much.
- **Flame graph** — a picture of a profile: each box is a function, its width is how much CPU it used, and boxes stack to show who called whom.
- **Continuous profiling** — sampling profiles all the time, cheaply, so you can compare today against yesterday. (Tools: Pyroscope, Parca — Day 78.)
- **Heap dump** — a frozen copy of everything in a process's memory, used to find leaks.
- **Thread dump / goroutine dump** — a frozen list of every thread and its stack, used to find freezes and deadlocks.
- **Core dump** — the whole process memory, saved for post-mortem debugging after a crash.
- **pprof** — Go's built-in profiler: `/debug/pprof/` serves CPU, heap, and goroutine profiles over HTTP, and `go tool pprof` reads them.

---

## 8. Common mistakes

1. **No deploy markers on your graphs.** The most common incident question is "what changed at 5:47 PM?" — and teams without events answer it from memory and chat history. One `curl` in your deploy script (Part A, Step 2) fixes this forever.
2. **Profiling only when things are already broken.** A profile taken during an incident shows what is hot *now* — but without a baseline from *before*, you cannot tell what changed. Continuous profiling (or even a weekly manual profile) gives you the "before" picture for free.
3. **Restarting before taking a dump.** The first instinct during a memory leak or a freeze is to restart the sick pod. Restarting destroys the evidence. Take the heap or thread dump first (it takes seconds), *then* restart. The dump is the only way to find the leak — otherwise it comes back next week, and you still will not know why.

---

## 9. Recap — the 5 things to remember

1. Events answer "what changed?"; profiles answer "which code?"; dumps answer "what was it holding when it died?" — three questions the pillars cannot answer.
2. Most incidents are caused by a change — put deploy events on your graphs automatically, and the first question of every incident answers itself.
3. A profile turns "the service is slow" into "this function uses 62% of CPU" — and a flame graph shows it in one picture.
4. A dump is frozen evidence: heap dumps find leaks, thread dumps find freezes — but only if you take the dump *before* you restart.
5. Add events first (cheap), profiles second (cheap once the agent is installed), and keep dumps as the emergency tool they are — heavy, manual, and full of secrets.

**Tomorrow — Day 6:** OpenTelemetry: the vendor-neutral standard — the one API that lets you send metrics, logs, and traces to any backend, and why the whole industry agreed on it.
