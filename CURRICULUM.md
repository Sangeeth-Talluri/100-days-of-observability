# 100 Days of Observability — The Full Roadmap

One topic a day, in a proper learning order. Each day has a theory part (explained in simple words, like to a smart five-year-old) and a technical implementation part (real commands, real config, real examples).

**Who this is for:** SREs, developers, and DevOps engineers who want to truly understand observability — not just install a tool, but know *how to use it, when to use it, and what problem it solves*.

Progress is tracked by the checkboxes. The daily lesson lives in `days/day-NNN-topic/`.

---

## Phase 1 — Foundations (Days 1–10): What observability really is

- [x] Day 1 — What is observability? (definition, and how it is different from monitoring)
- [x] Day 2 — Why observability matters: the unknown-unknowns problem
- [x] Day 3 — Telemetry: the data observability runs on
- [x] Day 4 — The three pillars: metrics, logs, traces (overview)
- [x] Day 5 — Signals beyond the pillars: events, profiles, and dumps
- [x] Day 6 — OpenTelemetry: the vendor-neutral standard
- [ ] Day 7 — Instrumentation: manual vs automatic
- [ ] Day 8 — The terms map: observability vs monitoring vs APM vs logging
- [ ] Day 9 — The observability lifecycle: collect → ship → store → query → visualize → alert
- [ ] Day 10 — Cardinality: the one concept that decides your power and your bill

## Phase 2 — Metrics (Days 11–25): Numbers that tell the truth

- [ ] Day 11 — Metrics deep dive: samples, labels, and time series
- [ ] Day 12 — Metric types: counter, gauge, histogram, summary
- [ ] Day 13 — Prometheus: architecture and the pull model
- [ ] Day 14 — PromQL basics: instant vectors, ranges, and rate()
- [ ] Day 15 — PromQL advanced: aggregations, joins, and recording rules
- [ ] Day 16 — Exporters: node_exporter and the exporter pattern
- [ ] Day 17 — Pushgateway, and push vs pull trade-offs
- [ ] Day 18 — Service discovery and relabeling in Prometheus
- [ ] Day 19 — Alertmanager: routing, grouping, and silences
- [ ] Day 20 — Writing good alerts: symptoms vs causes, RED and USE methods
- [ ] Day 21 — Long-term metric storage: Thanos, Cortex, and Mimir
- [ ] Day 22 — Prometheus federation
- [ ] Day 23 — Prometheus sharding and horizontal scaling
- [ ] Day 24 — Google Managed Service for Prometheus
- [ ] Day 25 — Amazon Managed Service for Prometheus and Azure Monitor managed Prometheus

## Phase 3 — Logs (Days 26–38): The story in words

- [ ] Day 26 — Logs deep dive: what makes a good log line
- [ ] Day 27 — Structured logging (JSON) and log levels done right
- [ ] Day 28 — Log aggregation architecture: agents, shippers, and sinks
- [ ] Day 29 — Loki: log aggregation, the Prometheus way
- [ ] Day 30 — Collecting logs with Promtail and Grafana Alloy
- [ ] Day 31 — LogQL: querying logs like metrics
- [ ] Day 32 — Elasticsearch and OpenSearch for logs
- [ ] Day 33 — Fluent Bit and Fluentd: log pipelines
- [ ] Day 34 — Log sinks: routing logs in GCP, AWS, and Azure
- [ ] Day 35 — Log sampling, filtering, and cost control
- [ ] Day 36 — Correlation: tying logs to traces and metrics
- [ ] Day 37 — Parsing and transforming logs in the pipeline
- [ ] Day 38 — Log-based metrics and log alerting

## Phase 4 — Traces (Days 39–50): Following one request end to end

- [ ] Day 39 — Distributed tracing: why microservices need it
- [ ] Day 40 — Spans, traces, and context propagation (W3C traceparent)
- [ ] Day 41 — Sampling strategies: head-based, tail-based, and why it matters
- [ ] Day 42 — Jaeger: the tracing backend basics
- [ ] Day 43 — Grafana Tempo: trace storage at scale, cheaply
- [ ] Day 44 — Zipkin and the tracing ecosystem story
- [ ] Day 45 — OpenTelemetry Collector: the pipeline engine
- [ ] Day 46 — OTLP: the protocol that moves telemetry
- [ ] Day 47 — Auto-instrumentation with OpenTelemetry SDKs
- [ ] Day 48 — eBPF and Beyla: zero-code instrumentation
- [ ] Day 49 — Odigos: the auto-instrumentation platform
- [ ] Day 50 — Service graphs and dependency maps from traces

## Phase 5 — Visualization and Dashboards (Days 51–60): Seeing it all

- [ ] Day 51 — Grafana and the single pane of glass idea
- [ ] Day 52 — Building your first dashboard: panels, layout, variables
- [ ] Day 53 — Dashboard design that actually helps during an incident
- [ ] Day 54 — Golden signals dashboards: latency, traffic, errors, saturation
- [ ] Day 55 — Explore mode: ad-hoc debugging across all signals
- [ ] Day 56 — Unified alerting in Grafana
- [ ] Day 57 — Dashboards as code: provisioning, Grafonnet, Terraform
- [ ] Day 58 — SLOs, SLIs, and error budgets
- [ ] Day 59 — Burn-rate alerts and SLO dashboards
- [ ] Day 60 — Single pane of glass: AWS + GCP + Azure + on-prem in one view

## Phase 6 — Cloud and Platform Observability (Days 61–75)

- [ ] Day 61 — AWS observability overview: CloudWatch, X-Ray, and friends
- [ ] Day 62 — The AWS managed stack: AMP and Amazon Managed Grafana
- [ ] Day 63 — ADOT: the AWS Distro for OpenTelemetry
- [ ] Day 64 — GCP observability overview: Cloud Monitoring, Logging, and Trace
- [ ] Day 65 — GCP log sinks and Log Analytics with BigQuery
- [ ] Day 66 — Google Managed Prometheus: production patterns
- [ ] Day 67 — Azure observability overview: Azure Monitor, Log Analytics, Application Insights
- [ ] Day 68 — KQL: querying logs and metrics in Azure
- [ ] Day 69 — Multi-cloud observability strategy
- [ ] Day 70 — On-prem and hybrid observability
- [ ] Day 71 — Kubernetes observability: the full kube-prometheus-stack
- [ ] Day 72 — Service mesh observability: Istio and Linkerd telemetry
- [ ] Day 73 — Serverless observability: Lambda, Cloud Run, Functions
- [ ] Day 74 — Edge and CDN observability
- [ ] Day 75 — Database observability: slow queries and exporter patterns

## Phase 7 — Advanced Practice (Days 76–90)

- [ ] Day 76 — High-cardinality debugging, done safely
- [ ] Day 77 — Exemplars: jumping from a metric spike to the exact trace
- [ ] Day 78 — Continuous profiling with Pyroscope and Parca
- [ ] Day 79 — Real user monitoring and synthetic checks
- [ ] Day 80 — Observability for CI/CD pipelines
- [ ] Day 81 — Incident response with observability: one worked timeline
- [ ] Day 82 — Capacity planning from metrics
- [ ] Day 83 — Cost optimization: telemetry budgets and tiered storage
- [ ] Day 84 — Telemetry pipelines at scale: buffering, backpressure, Kafka
- [ ] Day 85 — Security observability: audit logs and detection signals
- [ ] Day 86 — Observability for LLMs and AI agents
- [ ] Day 87 — AIOps: anomaly detection on telemetry
- [ ] Day 88 — Chaos engineering verified by observability
- [ ] Day 89 — Compliance and retention: telemetry as evidence
- [ ] Day 90 — Building an observability platform team

## Phase 8 — Capstone Builds (Days 91–100): Put it all together

- [ ] Day 91 — Capstone 1: instrument a sample microservice app, end to end
- [ ] Day 92 — Capstone 2: Prometheus and Grafana from zero on Kubernetes
- [ ] Day 93 — Capstone 3: a logs pipeline with Loki and Alloy
- [ ] Day 94 — Capstone 4: tracing with the OpenTelemetry Collector and Tempo
- [ ] Day 95 — Capstone 5: eBPF auto-instrumentation with Beyla and Odigos
- [ ] Day 96 — Capstone 6: SLOs and burn-rate alerts for a real service
- [ ] Day 97 — Capstone 7: one dashboard to rule the incident
- [ ] Day 98 — Capstone 8: multi-cloud telemetry into one backend
- [ ] Day 99 — Capstone 9: cut the telemetry bill by 50% without losing signal
- [ ] Day 100 — Your observability roadmap: the maturity model and what to learn next
