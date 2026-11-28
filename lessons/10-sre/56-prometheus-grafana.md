# Lesson 56. Prometheus, Grafana, Alertmanager

## Goal
Run a working monitoring stack, write PromQL that answers real questions, and create alerts that page only when users are affected.

## How Prometheus works

- **Pull model**: Prometheus **scrapes** HTTP `/metrics` endpoints of targets every N seconds.
- Targets come from static config or **service discovery** (Kubernetes, cloud APIs, Consul).
- Data is stored as time series: `metric_name{label="value",...}  value  timestamp`.
- **Exporters** translate other systems into metrics: `node_exporter` (Linux host), `blackbox_exporter` (probes HTTP/TCP/ICMP/DNS from outside), `postgres_exporter`, `redis_exporter`, `kube-state-metrics` (Kubernetes objects).
- Short-lived batch jobs push to the **Pushgateway** (only for that case).
- Long-term storage and HA: Thanos, Mimir, VictoriaMetrics (Prometheus-compatible).

## Run a lab stack

`compose.yaml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.55.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules.yml:/etc/prometheus/rules.yml:ro
    ports: ["9090:9090"]
  alertmanager:
    image: prom/alertmanager:v0.27.0
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports: ["9093:9093"]
  node-exporter:
    image: prom/node-exporter:v1.8.2
    pid: host
    volumes: ["/:/host:ro,rslave"]
    command: ["--path.rootfs=/host"]
  blackbox:
    image: prom/blackbox-exporter:v0.25.0
  grafana:
    image: grafana/grafana:11.3.0
    ports: ["3000:3000"]
```

`prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files: [/etc/prometheus/rules.yml]

alerting:
  alertmanagers:
    - static_configs: [{ targets: ["alertmanager:9093"] }]

scrape_configs:
  - job_name: prometheus
    static_configs: [{ targets: ["localhost:9090"] }]
  - job_name: node
    static_configs: [{ targets: ["node-exporter:9100"] }]
  - job_name: blackbox-http
    metrics_path: /probe
    params: { module: [http_2xx] }
    static_configs: [{ targets: ["https://example.com"] }]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

In Kubernetes, install **kube-prometheus-stack** with Helm: Prometheus Operator, Alertmanager, Grafana, node-exporter, kube-state-metrics and good default dashboards and alerts. Targets are declared with `ServiceMonitor` / `PodMonitor` objects.

## PromQL you will use every day

```promql
# is the target up?
up

# requests per second, per route, over the last 5 minutes
sum by (route) (rate(http_requests_total[5m]))

# error ratio (5xx / all)
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# p95 latency from a histogram, per route
histogram_quantile(0.95, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))

# CPU usage % per node
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))

# memory available %
100 * node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# disk will be full within 24 h (linear forecast from the last 6 h)
predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 24 * 3600) < 0

# certificate expires in less than 14 days
probe_ssl_earliest_cert_expiry - time() < 14 * 86400

# Kubernetes: containers restarting
increase(kube_pod_container_status_restarts_total[1h]) > 3

# Kubernetes: OOM kills
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

# CPU throttling ratio per container
sum by (pod, container) (rate(container_cpu_cfs_throttled_periods_total[5m]))
  / sum by (pod, container) (rate(container_cpu_cfs_periods_total[5m]))
```

Guidelines: always `rate()` a counter before summing; `sum by (...)` to aggregate away instance labels; range `[5m]` at least 4× the scrape interval.

**Recording rules** precompute expensive queries for dashboards and SLOs:

```yaml
groups:
  - name: recording
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

## Alerting rules

`rules.yml`:

```yaml
groups:
  - name: service
    rules:
      - alert: HighErrorRate
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (job) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels: { severity: page }
        annotations:
          summary: "{{ $labels.job }}: {{ $value | humanizePercentage }} of requests fail"
          runbook_url: https://example.com/runbooks/high-error-rate

  - name: host
    rules:
      - alert: DiskWillFillIn24h
        expr: predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 86400) < 0
        for: 30m
        labels: { severity: ticket }
        annotations:
          summary: "{{ $labels.instance }} {{ $labels.mountpoint }} will be full within 24h"

      - alert: TargetDown
        expr: up == 0
        for: 5m
        labels: { severity: ticket }
```

`for:` = condition must hold this long before firing; avoids alerts on one bad scrape.

## Alertmanager

Groups, deduplicates, silences and routes alerts to the right receivers.

`alertmanager.yml`:

```yaml
route:
  receiver: team-tickets
  group_by: [alertname, job]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: [severity="page"]
      receiver: oncall-pager

receivers:
  - name: oncall-pager
    pagerduty_configs:
      - routing_key: <secret>
  - name: team-tickets
    slack_configs:
      - api_url: <secret webhook>
        channel: "#alerts"

inhibit_rules:
  - source_matchers: [alertname="TargetDown"]
    target_matchers: [severity="ticket"]
    equal: [instance]
```

Silences (`amtool silence add ...` or the UI) for planned maintenance.

## Good alerting

- **Page on symptoms users feel** (error rate, latency, SLO burn: lesson 57), not on causes (CPU 90%). Causes go to dashboards and tickets.
- Every page must be **actionable** and have a **runbook** link.
- Two severities are enough to start: **page** (wake someone now) and **ticket** (look during working hours).
- Review alerts after every on-call shift: delete or fix noisy ones. Alert fatigue makes people ignore the alert that matters.
- Monitor the monitoring: a dead-man's switch alert (`vector(1)`) that must always fire; if it stops arriving, your alerting is broken.

## Grafana

- Add Prometheus (and Loki/Tempo) as data sources.
- Import community dashboards to start ("Node Exporter Full"), then build your own service overview.
- Use variables (`$job`, `$instance`) to reuse one dashboard for many services.
- Keep dashboards as code (JSON in Git, provisioning, or Grafana's Terraform provider).

## Practice

1. Start the lab stack. Open Prometheus, run `up`, then the CPU and memory queries.
2. Import a node exporter dashboard in Grafana.
3. Add the blackbox probe for a site you own and alert when the certificate expires in under 14 days.
4. Instrument your app (lesson 55) and graph RPS, error ratio and p95 latency per route.
5. Write the `HighErrorRate` alert, trigger it by making the app return 500s, and receive it in a Slack/Telegram/email receiver.
6. Use `predict_linear` with a file that grows (`fallocate` in a loop) and watch the alert fire before the disk is full.

## Check yourself

- Why must you `rate()` a counter before `sum`?
- What does `for: 10m` do in an alert rule?
- Why page on error rate rather than on CPU usage?

---

<p align="left">
  <a href="media/56-prometheus-grafana.mp4"><img width="640" src="media/56-prometheus-grafana.gif" alt="Lesson 56 animation: Prometheus, Grafana, Alertmanager"></a>
</p>

## Next
[Lesson 57. SLI, SLO, error budgets](57-slo-error-budgets.md)
