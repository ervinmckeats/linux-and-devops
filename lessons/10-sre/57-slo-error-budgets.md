# Lesson 57. SLI, SLO, error budgets

## Goal
Define reliability in numbers users care about, alert on it, and use it to balance new features against stability.

## Definitions

- **SLI** (service level indicator) — a measurement of user experience, usually a ratio of good events to all events.
  *"Share of HTTP requests to /api that return non-5xx in under 300 ms."*
- **SLO** (service level objective) — a target for an SLI over a window.
  *"99.9% of requests are good over 28 days."*
- **SLA** (service level agreement) — a contract with consequences (credits, penalties). SLA is looser than the internal SLO, so you notice and act before you owe money.
- **Error budget** — `1 − SLO`. The amount of failure you are **allowed**. At 99.9% over 28 days: 0.1% of requests, or about 40 minutes of total outage.

## Why not 100%?

- Users cannot tell 99.99% from 100% — their Wi-Fi, phone and ISP fail more often.
- Each extra nine costs roughly 10× more (redundancy, process, slower releases).
- 100% means you can never change anything. The error budget is permission to ship.

| SLO | Allowed downtime per 30 days |
|---|---|
| 99% | 7 h 12 min |
| 99.5% | 3 h 36 min |
| 99.9% | 43 min |
| 99.95% | 21 min |
| 99.99% | 4 min 19 s |

## Choosing SLIs

Measure as close to the user as possible (load balancer or edge, not inside the pod).

| Service type | SLI |
|---|---|
| Request/response API | **availability**: good responses / all; **latency**: requests faster than X / all |
| Data pipeline | **freshness**: data newer than X minutes; **correctness**; **coverage** |
| Storage | durability, read/write availability |
| Batch job | share of runs that finish on time |

Pick a few SLIs per **user journey** ("log in", "search", "checkout"), not per microservice.

## Writing an SLO

```yaml
service: checkout-api
owner: team-payments
slis:
  - name: availability
    good: http_requests_total{route="/checkout", status!~"5.."}
    total: http_requests_total{route="/checkout"}
    objective: 99.9%
    window: 28d
  - name: latency
    good: requests to /checkout faster than 500 ms
    total: all requests to /checkout
    objective: 99%
    window: 28d
excluded: synthetic monitoring traffic, requests rejected with 4xx
```

Start with what you achieve today (look at the last 4 weeks), then tighten if users need more.

## PromQL for SLOs

```promql
# availability SLI over 28 days
sum(rate(http_requests_total{route="/checkout",status!~"5.."}[28d]))
  / sum(rate(http_requests_total{route="/checkout"}[28d]))

# latency SLI: share of requests under 0.5 s (needs a histogram bucket at 0.5)
sum(rate(http_request_duration_seconds_bucket{route="/checkout",le="0.5"}[28d]))
  / sum(rate(http_request_duration_seconds_count{route="/checkout"}[28d]))

# error budget remaining (1 = untouched, 0 = spent)
1 - ((1 - availability_sli_28d) / (1 - 0.999))
```

Long windows are expensive to query; use recording rules (lesson 56) or a generator such as Sloth or Pyrra.

## Alerting on burn rate

**Burn rate** = how fast you spend the budget. Burn rate 1 spends exactly the whole budget in the SLO window; burn rate 14.4 spends 2% of a 30-day budget in one hour.

Multi-window, multi-burn-rate alerts (fast detection, few false alarms):

| Severity | Burn rate | Long window | Short window | Budget spent when it fires |
|---|---|---|---|---|
| page | 14.4 | 1 h | 5 min | 2% |
| page | 6 | 6 h | 30 min | 5% |
| ticket | 1 | 3 d | 6 h | 10% |

```yaml
- alert: CheckoutErrorBudgetFastBurn
  expr: |
    (
      sum(rate(http_requests_total{route="/checkout",status=~"5.."}[1h]))
        / sum(rate(http_requests_total{route="/checkout"}[1h])) > (14.4 * 0.001)
    )
    and
    (
      sum(rate(http_requests_total{route="/checkout",status=~"5.."}[5m]))
        / sum(rate(http_requests_total{route="/checkout"}[5m])) > (14.4 * 0.001)
    )
  labels: { severity: page }
  annotations:
    summary: "Checkout is burning its error budget 14x too fast"
    runbook_url: https://example.com/runbooks/checkout-slo
```

The short window makes the alert stop quickly once the problem is fixed.

These alerts replace most cause-based pages: you are woken up only when users are hurt at a rate that matters.

## Error budget policy

The SLO is useful only if it changes decisions. Agree in writing, with product owners:

- **Budget left** → ship features at normal speed; experiments allowed.
- **Budget nearly spent** → extra review for risky changes, prioritize reliability fixes.
- **Budget exhausted** → feature freeze for that service except reliability work, until the budget recovers; postmortem for big single burns (lesson 59).
- Budget consistently unused → the SLO may be too loose, or you can move faster and take more risk.

## Practice

1. Pick a service (your lab app is fine). Write two SLIs and SLOs in the YAML format above.
2. Implement the availability SLI and error-budget-remaining queries in Prometheus with recording rules.
3. Build a Grafana panel with SLI, SLO line and budget remaining.
4. Add the fast-burn alert; inject 5% errors for 10 minutes and watch it fire and resolve.
5. Write a one-page error budget policy and the people who must agree to it.

## Check yourself

- SLI vs SLO vs SLA?
- How much downtime does 99.9% allow per month?
- Why alert on burn rate instead of "error rate > 1%"?

## Next
[Lesson 58. Incident response and on-call](58-incident-response.md)
