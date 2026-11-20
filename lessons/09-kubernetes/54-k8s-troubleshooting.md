# Lesson 54. Kubernetes troubleshooting: CrashLoopBackOff, OOMKilled, Pending and friends

## Goal
Recognize every common pod failure by its status, find the cause in minutes, and fix it.

## The universal routine

```bash
kubectl get pods -n NS -o wide                  # STATUS, RESTARTS, NODE
kubectl describe pod POD -n NS                  # Events at the bottom; Last State; Reason; Exit Code
kubectl logs POD -n NS --previous               # logs of the crashed container
kubectl get events -n NS --sort-by=.lastTimestamp | tail -20
kubectl get deploy,rs -n NS                     # is the rollout stuck?
```

90% of answers are in `describe` → **Events** and **Last State**.

## Status → cause → fix

### Pending

The pod is not scheduled to any node.

| Event says | Cause | Fix |
|---|---|---|
| `0/5 nodes are available: insufficient cpu/memory` | requests too big or cluster full | lower requests, add nodes, check cluster autoscaler |
| `node(s) had untolerated taint` | taints | add toleration or use other nodes |
| `didn't match Pod's node affinity/selector` | wrong labels | fix selector or label nodes |
| `pod has unbound immediate PersistentVolumeClaims` | PVC not bound | check StorageClass, `kubectl describe pvc` |
| `Too many pods` | node pod limit | more nodes |

```bash
kubectl describe node NODE | grep -A8 "Allocated resources"
kubectl get pvc -n NS; kubectl describe pvc NAME -n NS
```

### ImagePullBackOff / ErrImagePull

| Message | Fix |
|---|---|
| `manifest unknown` / `not found` | wrong image name or tag |
| `unauthorized` / `pull access denied` | add `imagePullSecrets` or node registry permissions |
| `toomanyrequests` | registry rate limit (Docker Hub): use a mirror or authenticate |
| `no match for platform` | image built for amd64, node is arm64 (or reverse): build multi-arch |
| timeout | node cannot reach the registry: egress firewall, NAT, DNS |

### CrashLoopBackOff

The container starts, exits, Kubernetes restarts it with growing delay (10 s, 20 s, 40 s ... up to 5 min). It is a **symptom**; the reason is in `Last State` and the previous logs.

```bash
kubectl describe pod POD | grep -A6 "Last State"
kubectl logs POD --previous
```

| Exit code / reason | Likely cause |
|---|---|
| `1` / `Error` | app error: bad config, missing env var, cannot connect to DB at startup, migration failed |
| `127` | command not found: wrong `command`/`args`, missing binary in the image |
| `126` | not executable: permissions, wrong architecture |
| `137` / `OOMKilled` | memory limit exceeded (see below) |
| `137` without OOMKilled | killed by SIGKILL: often the **liveness probe** failed and kubelet killed it |
| `143` | SIGTERM: something stopped it (probe, rollout) |
| `0` / `Completed` | main process exited normally — a Deployment expects a long-running process |

Debug a container that dies too fast to exec into:

```bash
kubectl debug POD -it --copy-to=dbg --container=api -- sh    # copy of the pod with a shell
# or temporarily override the command in the Deployment:
#   command: ["sleep", "infinity"]
```

### OOMKilled

```
Last State:  Terminated
  Reason:    OOMKilled
  Exit Code: 137
```

The container exceeded its memory **limit** (a cgroup OOM, lesson 37); the node may have plenty of free memory.

1. Check limits vs usage: `kubectl top pod POD --containers`, memory graph over time.
2. Spike at start or under load → raise the limit to fit real peaks.
3. Slow steady growth to the limit → leak: profile the app; meanwhile restarts are the symptom.
4. Runtime does not know its limit → set `-XX:MaxRAMPercentage`, `GOMEMLIMIT`, `--max-old-space-size` below the limit.
5. Several containers in the pod (sidecars) → each has its own limit; check which one was killed.

Node-level memory pressure is different: the kubelet **evicts** pods (status `Evicted`, reason `The node was low on resource: memory`). BestEffort and Burstable pods over their requests go first. Fix requests so they reflect reality.

### Running but not Ready (0/1)

The readiness probe fails. Traffic is not sent to the pod.

```bash
kubectl describe pod POD | grep -A3 Readiness
kubectl exec POD -- wget -qO- localhost:8000/health/ready
```

Wrong port or path in the probe, app still warming up, or a dependency the readiness check needs is down.

### CreateContainerConfigError

A referenced ConfigMap/Secret or a key inside it does not exist. `describe` says which one.

### Terminating forever

```bash
kubectl get pod POD -o jsonpath='{.metadata.finalizers}'
```

Usually a finalizer waiting for a controller, or a node that is gone. Investigate before using `--force --grace-period=0`.

## Networking problems

```bash
kubectl get svc,endpointslices -n NS            # endpoints empty? → selector or readiness
kubectl run t --rm -it --image=nicolaka/netshoot --restart=Never -- bash
#   inside: dig api.shop.svc.cluster.local; curl -v http://api.shop/health; nc -zv db 5432
kubectl get networkpolicy -n NS                 # a default-deny you forgot?
kubectl -n kube-system logs -l k8s-app=kube-dns # CoreDNS errors
kubectl logs -n ingress deploy/ingress-nginx-controller | tail   # 502/503 from ingress
```

Ingress 502/503 usually means: Service has no ready endpoints, wrong `targetPort`, or the app listens on `127.0.0.1` instead of `0.0.0.0` inside the container.

## Node problems

```bash
kubectl get nodes                               # NotReady?
kubectl describe node NODE                      # Conditions: MemoryPressure, DiskPressure, PIDPressure
kubectl debug node/NODE -it --image=ubuntu      # then: chroot /host; journalctl -u kubelet; df -h; crictl ps
```

DiskPressure is often images and container logs filling `/var/lib/containerd` or `/var/log`.

## Rollout stuck

```bash
kubectl rollout status deploy/api -n NS
kubectl describe deploy api -n NS               # "ProgressDeadlineExceeded"
kubectl get rs -n NS                            # new ReplicaSet pods failing?
kubectl rollout undo deploy/api -n NS           # restore service first, investigate after
```

## Cheat sheet

| Status | First command | Most common cause |
|---|---|---|
| Pending | `describe pod` → Events | requests too big / no capacity / PVC |
| ImagePullBackOff | `describe pod` | wrong tag, no pull secret |
| CrashLoopBackOff | `logs --previous` | app config error at startup |
| OOMKilled | `describe` Last State, `top` | limit below real usage, leak |
| 0/1 Ready | `describe` probe section | wrong probe path/port |
| CreateContainerConfigError | `describe` | missing ConfigMap/Secret key |
| Evicted | `describe node` | node memory/disk pressure |

## Practice

Create each failure on purpose in a local cluster, then diagnose it using only the commands above:

1. Request `cpu: 64` → Pending.
2. Image tag `nginx:does-not-exist` → ImagePullBackOff.
3. `command: ["false"]` → CrashLoopBackOff with exit code 1.
4. A pod that allocates more than its 64Mi limit → OOMKilled.
5. Readiness probe on a wrong port → 0/1 Ready, empty endpoints.
6. Reference a Secret that does not exist → CreateContainerConfigError.
7. Service selector typo → ingress returns 503.

Write a one-line "symptom → cause → fix" note for each in your own runbook.

## Check yourself

- CrashLoopBackOff is a symptom. Where do you find the actual cause?
- Exit code 137: how do you tell OOMKilled from a liveness-probe kill?
- A Service has no endpoints. Name two possible reasons.

## Next
[Lesson 55. Observability: metrics, logs, traces](../10-sre/55-observability.md)
