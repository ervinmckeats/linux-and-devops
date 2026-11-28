# Lesson 52. Requests, limits, probes, autoscaling

## Goal
Make workloads schedule predictably, stay healthy, survive node maintenance and scale with load.

## Requests and limits

```yaml
resources:
  requests:
    cpu: 250m        # 0.25 CPU - used for SCHEDULING
    memory: 256Mi
  limits:
    memory: 512Mi    # hard cap - exceed it and the container is OOMKilled
    # cpu limit: often left out on purpose, see below
```

- **Requests**: what the scheduler reserves on a node. A pod is placed only on a node with enough unreserved capacity. Requests also set the CPU share under contention.
- **Memory limit**: enforced by the cgroup (`memory.max`, lesson 38). Exceeding it → kernel OOM-kills the container → `OOMKilled`, exit code 137, restart.
- **CPU limit**: enforced by throttling (`cpu.max`). The container is paused when its quota for the period is used. Average CPU looks fine, latency suffers. Many teams set CPU requests and **no CPU limit** for latency-sensitive services; others keep limits for strict multi-tenant fairness. Measure throttling (`container_cpu_cfs_throttled_periods_total`) before deciding.

### QoS classes

| Class | When | Evicted under node pressure |
|---|---|---|
| Guaranteed | requests = limits for CPU and memory in every container | last |
| Burstable | some requests set | middle |
| BestEffort | nothing set | first |

### Sizing

1. Start from observed usage: `kubectl top pods`, or p95 memory and CPU from Prometheus over a week.
2. Memory request ≈ normal usage, memory limit = request + safe headroom for peaks.
3. CPU request ≈ typical usage under normal load.
4. Runtimes need to know their limit: JVM (`-XX:MaxRAMPercentage=75`), Go (`GOMEMLIMIT`), Node (`--max-old-space-size`).
5. Enforce defaults with `LimitRange` so nobody deploys BestEffort pods by accident.

## Probes

```yaml
startupProbe:            # gives slow starters time; other probes wait until it succeeds
  httpGet: { path: /health/started, port: 8000 }
  periodSeconds: 5
  failureThreshold: 30   # up to 150 s to start
readinessProbe:          # "can I receive traffic now?"  fail → removed from Service endpoints
  httpGet: { path: /health/ready, port: 8000 }
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:           # "am I stuck?"  fail → container is restarted
  httpGet: { path: /health/live, port: 8000 }
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 2
```

Rules:
- **Readiness** may check dependencies the pod needs to serve (DB connection pool ready, cache warm).
- **Liveness** must check only the process itself. If liveness checks the database, a DB blip restarts **every** pod at once and turns a small incident into an outage.
- A liveness probe that is too aggressive on a slow app causes restart loops under load. Use a startup probe for slow starts instead of a huge `initialDelaySeconds`.
- Probe types: `httpGet`, `tcpSocket`, `exec`, `grpc`.

## Graceful shutdown

On delete/rollout: pod is removed from endpoints **and** gets `SIGTERM` at about the same time, then `SIGKILL` after `terminationGracePeriodSeconds` (default 30).

```yaml
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: api
      lifecycle:
        preStop:
          exec: { command: ["sleep", "5"] }   # let load balancers stop sending traffic first
```

The app must handle `SIGTERM`: stop accepting, finish in-flight requests, close connections, exit.

## Availability during maintenance

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api, namespace: shop }
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: api }
```

`kubectl drain` and cluster upgrades respect PDBs. Also spread replicas so one node or zone failure does not take them all:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels: { app: api }
```

## Scheduling controls

- `nodeSelector` / node **affinity**: run only on certain nodes (GPU, arm64, dedicated pool).
- **Taints and tolerations**: keep everyone off a node except pods that tolerate the taint.
- Pod **anti-affinity**: do not put two replicas on the same node.
- `priorityClassName`: important pods can preempt less important ones when the cluster is full.

## Autoscaling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api, namespace: shop }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }   # % of REQUESTS
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

- Utilization is relative to **requests**: without requests the HPA cannot work.
- Needs metrics-server. For custom metrics (requests per second, queue length) use Prometheus Adapter or **KEDA**.
- Do not set `replicas` in the Deployment YAML when an HPA manages it, or every apply resets it.

### Other autoscalers

- **VPA** (Vertical Pod Autoscaler): recommends or sets requests. Use in recommendation mode to size workloads.
- **Cluster Autoscaler / Karpenter**: add nodes when pods are `Pending` for lack of capacity, remove empty nodes.

## Practice

1. Deploy an app with no requests/limits and check its QoS class (`kubectl get pod -o jsonpath='{.status.qosClass}'`). Then make it Guaranteed.
2. Set a memory limit of 64Mi on a pod that allocates 200Mi. Watch `OOMKilled` in `describe` and the restart counter.
3. Add a liveness probe that checks a dependency; stop the dependency and watch every pod restart. Then fix the probe design.
4. Create a PDB with `minAvailable: 2` for 3 replicas and drain a node.
5. Create an HPA on CPU, generate load (`kubectl run load --image=busybox -- sh -c 'while true; do wget -qO- http://api; done'`) and watch scaling with `kubectl get hpa -w`.

## Check yourself

- What does the scheduler use: requests or limits?
- What happens when a container exceeds its memory limit? Its CPU limit?
- Why must a liveness probe not check the database?

---

<p align="left">
  <a href="media/52-resources-probes-autoscaling.mp4"><img width="640" src="media/52-resources-probes-autoscaling.gif" alt="Lesson 52 animation: Requests, limits, probes, autoscaling"></a>
</p>

## Next
[Lesson 53. Helm and GitOps](53-helm-gitops.md)
