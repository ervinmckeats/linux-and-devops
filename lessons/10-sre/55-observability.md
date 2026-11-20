# Lesson 55. Observability: metrics, logs, traces

## Goal
Know what to measure and collect so you can answer "is it broken, where, and why?" without logging in to servers.

## Monitoring vs observability

- **Monitoring** answers known questions: "is CPU above 90%?", "is the site up?"
- **Observability** lets you answer **new** questions about behaviour you did not predict, from the data the system already emits.

You need both. They are built from three main signals.

## The three signals

| Signal | What | Good for | Cost |
|---|---|---|---|
| **Metrics** | numbers over time with labels: `http_requests_total{status="500",route="/api"}` | dashboards, alerts, trends, SLOs | cheap per data point, but watch label **cardinality** |
| **Logs** | timestamped events with details | "what exactly happened in this request" | expensive at volume |
| **Traces** | one request's path through all services, with timings per step (spans) | "which service made it slow" | sampled to control cost |

Plus **profiles** (continuous CPU/memory profiling) and **events** (deploys, config changes): put deploy markers on your dashboards; most incidents follow a change.

They work best **linked**: an alert on a metric → a dashboard → an exemplar trace for a slow request → the logs of that trace by `trace_id`.

## What to measure

### Golden signals (for every user-facing service)

1. **Latency**: time to serve a request; separate successful from failed requests; use percentiles (p50, p95, p99), not averages.
2. **Traffic**: requests per second.
3. **Errors**: rate of failed requests (5xx, timeouts, wrong answers).
4. **Saturation**: how full the service is: CPU, memory, queue depth, connection pool, thread pool.

**RED** for services: Rate, Errors, Duration. **USE** for resources: Utilization, Saturation, Errors (lesson 39).

### Layers

| Layer | Examples |
|---|---|
| User experience | synthetic checks from outside, real user monitoring, SLIs (lesson 57) |
| Service | RED metrics per endpoint, dependency latency, queue lag |
| Platform | Kubernetes: restarts, pending pods, OOM kills, HPA status, node conditions |
| Infrastructure | node CPU/memory/disk/network, disk fill forecast, certificate expiry |
| Business | orders per minute, sign-ups, payments: often the first sign of a silent failure |

## Metrics basics

Metric types (Prometheus naming):
- **Counter**: only goes up (`requests_total`). You look at its rate: `rate(requests_total[5m])`.
- **Gauge**: goes up and down (`memory_bytes`, `queue_length`).
- **Histogram**: counts in buckets (`request_duration_seconds_bucket`) → percentiles across many instances.
- **Summary**: percentiles computed in the client; cannot be aggregated across instances.

**Cardinality** = number of unique label combinations. `user_id` or full URL as a label creates millions of series and kills your metrics system. Labels should have bounded values: route template (`/users/:id`), status code, method, region.

## Logs done right

- **Structured** (JSON), one event per line, to stdout in containers.
- Always include: timestamp (UTC, ISO 8601), level, service, version, `trace_id`, request ID, and the key fields of the event.
- Levels mean something: `ERROR` = needs attention, `WARN` = unusual but handled, `INFO` = business events, `DEBUG` = off in prod by default.
- Never log secrets, tokens, passwords, full card numbers or unnecessary personal data.
- Centralize them: an agent (Fluent Bit, Vector, Promtail/Alloy, OpenTelemetry Collector) ships logs to a store (Loki, Elasticsearch/OpenSearch, cloud logging). Set **retention** deliberately.

```json
{"ts":"2026-01-10T12:00:01.123Z","level":"error","service":"api","version":"1.4.2","trace_id":"4bf92f3577b34da6","route":"/orders","status":502,"duration_ms":3012,"msg":"payment provider timeout"}
```

## Traces

A **trace** is a tree of **spans**. Each span: service, operation, start, duration, attributes, status. Context (trace ID) is propagated between services in HTTP headers (`traceparent`, W3C standard).

```
GET /checkout                             1200 ms
├── api: validate cart                      15 ms
├── api → inventory: reserve               80 ms
└── api → payments: charge               1050 ms   ← here
    └── payments → bank API               1020 ms
```

Use **OpenTelemetry** (OTel): one vendor-neutral standard and SDKs for traces, metrics and logs, plus the **Collector** to receive, process and export them to any backend (Jaeger, Tempo, Zipkin, commercial vendors). Auto-instrumentation covers HTTP frameworks, DB clients and message queues with little code.

## A typical open-source stack

```
apps (OTel SDK, /metrics)        node_exporter, kube-state-metrics, cAdvisor
         │                                      │
         ▼                                      ▼
  OpenTelemetry Collector  ───────────▶  Prometheus (metrics) ──▶ Alertmanager ──▶ on-call
         │                               Loki (logs)
         └────────────────────────────▶  Tempo / Jaeger (traces)
                                                │
                                         Grafana (dashboards, explore, correlate)
```

Managed alternatives exist in every cloud and from vendors; the concepts are the same.

## Dashboards that help during an incident

- One **overview** dashboard per service: golden signals + SLO status + recent deploys.
- Drill-down dashboards for dependencies and resources.
- Same time range and layout conventions everywhere. Top-left = most important.
- Fewer, better panels. A wall of 60 graphs is hard to read during an incident.

## Practice

1. Add structured JSON logging with a request ID to the small app from earlier lessons.
2. Expose RED metrics from the app with a Prometheus client library (`/metrics`).
3. Add OpenTelemetry auto-instrumentation, run Jaeger locally (`docker run -p 16686:16686 -p 4317:4317 jaegertracing/all-in-one`), and look at a trace.
4. Make one request slow on purpose (sleep in a downstream call) and find it from the trace.
5. List the golden signals for a service you know and where you would get each one.

## Check yourself

- What are the four golden signals?
- Why are averages bad for latency?
- Why is `user_id` a bad metric label but a good log/trace attribute?

## Next
[Lesson 56. Prometheus, Grafana, Alertmanager](56-prometheus-grafana.md)
