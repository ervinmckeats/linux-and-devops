# Lesson 60. Reliability patterns and capacity planning

## Goal
Design and review systems so that partial failures stay partial, and know you have enough capacity before users tell you.

## Failure is normal

In a system with many parts something is always broken: a node, a disk, a dependency, a network link. Reliability comes from **containing** failures, not from preventing all of them.

## Patterns

### Timeouts: always

Every network call needs a timeout. Without one, a slow dependency holds threads/connections until your service runs out and fails too.

- Set them explicitly; library defaults are often infinite or 30–60 s.
- Timeout < caller's timeout. If the user's request has 2 s, inner calls get less.
- Pass a **deadline** down the call chain (gRPC does this natively).

### Retries: carefully

Retries fix brief glitches and can also multiply load during an outage (**retry storm**): 3 layers × 3 retries = 27× traffic to the failing service.

- Retry only **idempotent** operations (or use idempotency keys for payments).
- **Exponential backoff with jitter**: `sleep = random(0, base * 2^attempt)`, capped.
- Limit attempts (2–3) and use a **retry budget** (e.g. retries ≤ 10% of requests).
- Retry at one layer only, usually closest to the caller.
- Do not retry on 4xx (except 429 with `Retry-After`).

### Circuit breaker

After repeated failures to a dependency, **stop calling it** for a while and fail fast (or use a fallback). Periodically let a test request through (half-open) and close the circuit when it succeeds. Protects both you and the struggling dependency.

### Bulkheads

Isolate resources so one bad dependency cannot consume everything: separate connection pools/thread pools per dependency, separate worker queues per tenant or priority, separate node pools for critical workloads.

### Load shedding and rate limiting

When overloaded, reject some work quickly (`429`/`503`) instead of serving everyone slowly. Prioritize: health checks and paying users first, background jobs last. Rate-limit per client at the edge (lesson 27).

### Graceful degradation

Return something useful when a non-critical dependency fails: cached data, default recommendations, hide the reviews widget. Decide **in advance** which features are optional.

### Queues

Decouple producers from consumers; absorb spikes. Monitor **queue lag/age**, not only length. Use dead-letter queues for poison messages; make consumers idempotent (messages can arrive twice).

### Redundancy and failure domains

- N+1 (or N+2) instances: losing one still leaves enough capacity.
- Spread across **availability zones** (lesson 47); for critical systems, across regions.
- Avoid hidden single points of failure: one NAT gateway, one DNS provider, one secret store, one CI runner, one person who knows how it works.

### Safe change

Most incidents follow a change, so make change safe: canary and progressive delivery (lesson 44), feature flags, automatic rollback on SLO burn, backward-compatible migrations, one change at a time.

### Health checks with care

Readiness vs liveness (lesson 52). A load balancer health check that depends on a shared dependency can mark **all** instances unhealthy at once and turn a partial failure into a full outage.

## Testing reliability

- **Load tests** before launches and regularly: find the breaking point (k6, Locust, wrk).
- **Chaos engineering**: inject failures on purpose (kill pods, add latency, drop a zone) in staging, then carefully in production, with a hypothesis and a stop button. Tools: Chaos Mesh, Litmus, AWS FIS, `tc netem`:

```bash
sudo tc qdisc add dev eth0 root netem delay 200ms 50ms loss 1%
sudo tc qdisc del dev eth0 root
```

- `sudo tc qdisc add dev eth0 root netem delay 200ms 50ms loss 1%`: add latency and loss
- `sudo tc qdisc del dev eth0 root`: remove

- **Disaster recovery drills**: restore from backup, fail over a region, rebuild a cluster from Git. Measure real RTO/RPO (lesson 21).

## Capacity planning

Goal: enough capacity for expected peaks plus failures, at a reasonable cost.

1. **Find the unit of load**: requests per second, active users, messages per second, GB stored.
2. **Measure cost per unit**: from load tests and production metrics: e.g. one pod handles 400 RPS at p99 < 200 ms using 0.8 CPU and 600 MB.
3. **Forecast demand**: historical growth, seasonality (daily/weekly peaks, sales events), planned launches and marketing.
4. **Add headroom**: survive losing one AZ (N+1 per zone) and sudden spikes. A common target is peak utilization ≤ 60–70% of the safe limit.
5. **Know the limits that are not CPU**: database connections, disk IOPS, network bandwidth, cloud quotas (instances, IPs, load balancers), third-party API rate limits, licence limits.
6. **Automate what you can**: HPA and cluster autoscaling (lesson 52) for fast changes; planned increases for slow ones (databases, quotas, reserved capacity).
7. **Review** monthly or quarterly with graphs: usage vs capacity, forecast, cost.

Example:

```
Peak today: 6 000 RPS, growth 8%/month, Black Friday ×3 expected.
One pod: 400 RPS at target latency.
Black Friday in 4 months: 6 000 × 1.08^4 × 3 ≈ 24 500 RPS → 62 pods.
Survive one of 3 zones lost: 62 × 3/2 = 93 pods.
Headroom 30%: ~120 pods at peak. Today's maximum (HPA max) is 40 → raise max,
check node pool limits, cloud quotas and DB connection limits (120 pods × pool of 10 = 1 200 connections!).
```

## Design review checklist

- [ ] Every outbound call has a timeout; retries are bounded with backoff and jitter
- [ ] What happens when each dependency is slow? Down? Returns garbage?
- [ ] Which features degrade gracefully, which are critical?
- [ ] Single points of failure (infrastructure, data, people)?
- [ ] Runs in ≥ 2 zones; survives losing one at peak
- [ ] Capacity numbers known: per-instance limit, peak forecast, headroom, non-CPU limits
- [ ] SLOs defined, dashboards and burn-rate alerts exist, runbooks linked
- [ ] Deploys are progressive and roll back automatically
- [ ] Backups restore-tested; RTO/RPO measured
- [ ] Data migrations are backward compatible

## Practice

1. Add a timeout, bounded retries with jittered backoff and a simple circuit breaker to the HTTP client in your lab app.
2. Use `tc netem` to add 500 ms latency to the database host. Observe your service with and without timeouts.
3. Load-test your app to find the RPS where p99 latency breaks your SLO. Compute pods needed for 3× today's traffic with one zone down.
4. Kill a random pod every minute during a load test (simple chaos) and check that the SLO holds.
5. Run the design review checklist on a system you know and write down three risks.

## Check yourself

- Why can retries make an outage worse, and what limits that?
- Circuit breaker vs bulkhead: what does each protect against?
- Name three capacity limits that are not CPU or memory.

## Next
[Lesson 61. Troubleshooting playbook](../11-practice/61-troubleshooting-playbook.md)
