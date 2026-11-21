# Day 2 — Why Observability Matters: The Unknown-Unknowns Problem

**In one line:** The failures that hurt you most are the ones nobody predicted — observability exists so you can still find them.

---

## 1. Explain it like I'm five

Imagine you are a doctor in a small town.

**The easy patients** come in and say, "I have a fever." You know this problem. You have a thermometer, you know the medicine, you have seen it a hundred times. You planned for it.

**The hard patient** comes in and says, "I feel fine, but my left foot is cold." No fever. No pain. No test you already planned says "this is wrong." The problem is real — blood is not reaching the foot — but none of your normal checks can see it. If you only look at the things you already know to check, this patient goes home, and the foot gets worse.

That second patient is the **unknown-unknown**: a problem you did not know existed, so you never built a check for it.

Monitoring is great at the first patient. Observability exists for the second one.

---

## 2. The real definition: three kinds of problems

In 2002, US Secretary of Defense Donald Rumsfeld described three kinds of knowledge in a famous press briefing:

> "There are known knowns; there are things we know we know. We also know there are known unknowns; that is to say we know there are some things we do not know. But there are also unknown unknowns — the ones we don't know we don't know."

The same three boxes apply to your system:

| Box | What it means | Example |
|---|---|---|
| **Known-knowns** | You know the problem, and you know how to check for it | Disk full. A service is down. CPU at 100%. |
| **Known-unknowns** | You know the question, but not the answer yet | "How slow is checkout right now?" You have a dashboard, you just need to look. |
| **Unknown-unknowns** | You don't even know the question exists | A bug that sat quietly in your code for a month, waiting for one rare customer action to wake it up. |

**Monitoring covers the first two boxes.** You write a check for disk space, you build a dashboard for checkout latency, and those tools answer you. That work matters — never skip it.

**But unknown-unknowns cannot be checked in advance, by definition.** You cannot write an alert for a failure mode you have never imagined. So when one strikes, your dashboards say "everything is fine" while your users say "nothing works."

Observability is the answer to that box. It does not mean predicting the future. It means your system records such **rich, detailed, connected evidence** while it runs — every request tagged with who, where, what, and how long — that when a brand-new failure appears, you can ask a brand-new question and get an answer from data you already collected.

**Three facts make unknown-unknowns so dangerous:**

1. **They are new.** The last incident's fix does not cover them. Teams that only prepare for the last incident keep getting surprised.
2. **They hide in combinations.** One factor alone looks fine. The failure only happens when three things line up at once — one provider, one region, one kind of order. No single metric shows it.
3. **They are found late.** The first alarm is often a customer, a tweet, or a lost sale — not your monitoring. Every minute between "users notice" and "we understand" is damage.

That gap — between *something is wrong* and *we know what it is* — is called **mean time to detect (MTTD)** plus **mean time to understand**. Observability shrinks both. It does not stop new failures from happening. It stops them from staying mysterious for hours.

---

## 3. A real example: the day one customer broke the internet

On **June 8, 2021**, the CDN provider Fastly took down a huge part of the internet for about an hour — Amazon, Reddit, Twitch, The Guardian, The New York Times, and many more went dark at the same time.

The cause, in Fastly's own words (from their public incident summary):

> "We experienced a global outage due to an undiscovered software bug that surfaced on June 8 when it was triggered by a valid customer configuration change."

Read that sentence carefully, because it is the perfect unknown-unknown:

- **May 12, 2021** — Fastly shipped a software deployment. It contained a bug. Nobody knew. All tests passed. The bug was *dormant* — it did nothing wrong for 27 days.
- **June 8, 09:47 UTC** — one single customer pushed a **valid** configuration change. Not a mistake. Not an attack. A normal, allowed change that happened to include the exact conditions that woke the bug up.
- **Result** — 85% of Fastly's network started returning errors. Within one minute their monitoring spotted the disruption, and within 49 minutes 95% of the network was back — but only because they could *find* the trigger fast. The bug itself could never have been predicted by a dashboard: no check can alert on "a bug we don't know exists, waiting for a config we haven't seen yet."

That is the honest truth about unknown-unknowns: **the world's best engineering teams ship them too.** The difference between a 49-minute outage and a 6-hour outage is not "did you predict it?" — it is "when it happened, could your system explain itself?"

Now compare with the small example from Day 1 — the checkout that is slow only for Provider B, in one region, for orders above $100. Same shape, smaller scale: three conditions must line up, and a top-level "checkout is slow" graph hides the answer. That is the shape of almost every unknown-unknown. Section 7 lets you hunt one yourself.

---

## 4. What problem does it solve?

1. **Mystery incidents.** "We still don't know what happened" stops being the normal ending of an incident review. With rich telemetry, a new failure is a new question, not a dead end.
2. **The "works for me" trap.** A failure that hits only some users looks like "no problem" from the office. Slicing data by user, region, provider, or version turns "some users" into a precise list.
3. **Slow detection.** Unknown-unknowns are usually found by users first. Good observability means the system itself gives you the trail to follow the moment someone reports a symptom — so the time from "first report" to "root cause" drops from hours to minutes.
4. **Fighting the last war.** Teams that only add a check after each incident end up with a thousand checks and still get surprised. Observability invests in *question-asking power* instead of *one more check* — and that power covers the next incident too, whatever it is.

---

## 5. When does this matter most? (and when it doesn't)

**Unknown-unknowns matter most when:**
- Your system has many moving parts (services, queues, regions, providers) — combinations multiply, and so do the surprises.
- Failures are expensive — lost sales, broken trust, or pages at 3 AM.
- You deploy often — every release is a chance to plant a quiet bug like Fastly's.
- Your users are diverse — different regions, devices, plans, and providers mean a failure can hide in one slice.

**You can worry about this less when:**
- One small app, one server, few users — there are so few combinations that surprises are rare and easy to spot by hand.
- Downtime costs almost nothing — a hobby project can be inspected directly.

**Honest rule:** you cannot eliminate unknown-unknowns. You can only buy the ability to *find them fast*. That ability is what the rest of this series builds, one tool at a time.

---

## 6. Hands-on: hunt a hidden failure in 500 fake requests (20 minutes)

Theory is cheap. Now find an unknown-unknown yourself. You need only **Python 3** — nothing to install.

**Step 1 — Generate the incident.** Save this as `generate_incident.py` and run it. It creates 500 checkout requests as JSON lines. Most are healthy. A hidden failure is buried inside — and nothing in the totals will point at it.

```python
#!/usr/bin/env python3
"""Generate 500 fake checkout requests as JSON lines.

Most requests are healthy. A hidden failure is buried inside:
only one combination of provider + region + amount is slow or failing.
Nothing in the total error rate tells you that -- you must slice.
"""
import json
import random

random.seed(42)

providers = ["provider-a", "provider-b"]
regions = ["us-east", "eu-west"]

for i in range(500):
    provider = random.choice(providers)
    region = random.choice(regions)
    amount = random.choice([25, 60, 120, 250])
    latency_ms = random.randint(80, 400)
    status = 200
    # THE HIDDEN FAILURE (do not read this block until after step 2):
    if provider == "provider-b" and region == "eu-west" and amount > 100:
        latency_ms = random.randint(6000, 9000)
        status = 504 if random.random() < 0.5 else 200
    print(json.dumps({
        "request_id": f"req-{i:04d}",
        "provider": provider,
        "region": region,
        "amount_usd": amount,
        "latency_ms": latency_ms,
        "status": status,
    }))
```

```bash
python3 generate_incident.py > checkout.log
wc -l checkout.log
```

Expected output:

```
500 checkout.log
```

**Step 2 — Ask the "monitoring" question: how bad is it overall?**

```bash
python3 -c "
import json
rows=[json.loads(l) for l in open('checkout.log')]
print('total:', len(rows))
print('errors:', sum(1 for r in rows if r['status']>=500))
print('avg latency:', sum(r['latency_ms'] for r in rows)//len(rows), 'ms')
"
```

Expected output:

```
total: 500
errors: 34
avg latency: 1094 ms
```

This is the monitoring view: "6.8% of requests fail, average latency is over a second." You know *something* is wrong. You know nothing about *what*. No alert you pre-built could tell you more — the failure mode is unknown.

**Step 3 — Ask a new question: slice the data.** This is the observability move. Slice by provider, region, and amount — three dimensions at once:

```bash
python3 -c "
import json, collections
rows=[json.loads(l) for l in open('checkout.log')]
g=collections.defaultdict(lambda:[0,0,0])
for r in rows:
    k=(r['provider'], r['region'], 'over100' if r['amount_usd']>100 else 'under100')
    g[k][0]+=1
    if r['status']>=500: g[k][1]+=1
    g[k][2]+=r['latency_ms']
for k,(n,e,lat) in sorted(g.items()):
    print(k, 'errors:', e, '/', n, f'({100*e/n:.1f}%)', 'avg latency:', lat//n, 'ms')
"
```

Expected output:

```
('provider-a', 'eu-west', 'over100')  errors: 0 / 66  (0.0%)  avg latency: 253 ms
('provider-a', 'eu-west', 'under100') errors: 0 / 46  (0.0%)  avg latency: 229 ms
('provider-a', 'us-east', 'over100')  errors: 0 / 62  (0.0%)  avg latency: 257 ms
('provider-a', 'us-east', 'under100') errors: 0 / 66  (0.0%)  avg latency: 243 ms
('provider-b', 'eu-west', 'over100')  errors: 34 / 57 (59.6%) avg latency: 7662 ms
('provider-b', 'eu-west', 'under100') errors: 0 / 64  (0.0%)  avg latency: 247 ms
('provider-b', 'us-east', 'over100')  errors: 0 / 72  (0.0%)  avg latency: 245 ms
('provider-b', 'us-east', 'under100') errors: 0 / 67  (0.0%)  avg latency: 260 ms
```

**There it is.** One line is different from all the others: provider-b, eu-west, orders over $100 — 59.6% failing, 7.6 seconds average latency. Every other slice is healthy. The failure needed **three conditions at once**. The overall 6.8% error rate hid it completely.

Notice what made this possible: each log line carried **dimensions** — provider, region, amount — that let you ask a question nobody planned in advance. That is the whole game. A log line with only "error: 504" could never have answered it. (Day 10, cardinality, explains the price of adding dimensions — and why it is worth paying.)

**Step 4 — Try it on your own system.** Take one real incident from your team's past month. Sort it into a box:
- Known-known? You had a check for it and it fired.
- Known-unknown? You had the data, you just had to look.
- Unknown-unknown? Nobody saw it coming; the first sign was a user or a surprise.

If more than half your incidents land in the third box, your system is telling you what to invest in: richer telemetry, not more alerts.

---

## 7. Key terms from today

- **Known-knowns** — problems you know about, with a check already built for them. Monitoring's home turf.
- **Known-unknowns** — questions you know to ask, but haven't answered yet. A dashboard answers these.
- **Unknown-unknowns** — failures you did not know could happen, so no check exists for them. The reason observability exists.
- **MTTD** — mean time to detect: how long from "it broke" to "we know it broke."
- **MTTR** — mean time to resolution: how long an incident lasts, from detection to fix. (From Day 1.)
- **Slicing (or drilling down)** — splitting data by dimensions (region, provider, version, user type) until the bad slice stands out.
- **Dimension / label** — a tag on telemetry (like `provider="provider-b"`) that lets you slice and filter later.

---

## 8. Common mistakes

1. **"We have 200 alerts, so we are covered."** Two hundred checks for *known* problems still leave the unknown box empty. Alerts catch what you imagined. Observability catches what you didn't.
2. **Only adding a check after each incident.** That is fighting the last war. The next failure will be new — invest in data that answers new questions, not in a longer list of old ones.
3. **Logging without dimensions.** "Error: timeout" a thousand times is noise. "Error: timeout, provider=b, region=eu-west, amount=120" is evidence. If your logs can't be sliced, they can't find unknown-unknowns.

---

## 9. Recap — the 5 things to remember

1. Failures come in three boxes: known-knowns, known-unknowns, and unknown-unknowns — and the third box causes the worst incidents.
2. Monitoring handles the first two boxes. It cannot, by definition, cover the third.
3. Unknown-unknowns hide in combinations — one provider, one region, one kind of request — so averages and totals will not show them.
4. Observability means collecting rich, connected, sliceable data *before* the incident, so a brand-new question still gets an answer.
5. You cannot predict the next failure. You can buy the ability to find it in minutes instead of hours — that is the whole point of this series.

**Tomorrow — Day 3:** Telemetry: the data observability runs on — what it is, where it comes from, and what good telemetry looks like.
