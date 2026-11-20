# Lesson 51. Deployments, Services, Ingress, ConfigMaps, Secrets

## Goal
Deploy a real application: configured, reachable from outside over HTTPS, updated with zero downtime, with persistent storage where needed.

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: shop
  labels: { app: api }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app: api }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # at most 1 extra pod during the update
      maxUnavailable: 0    # never below 3 ready pods
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: ghcr.io/example/api:1.4.2
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef: { name: api-config }
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: api-db, key: password }
          volumeMounts:
            - name: config
              mountPath: /etc/api
              readOnly: true
          # requests/limits and probes: lesson 52
      volumes:
        - name: config
          configMap:
            name: api-config-files
```

A rolling update starts new pods, waits until they are **Ready**, then removes old ones. Without a readiness probe "ready" just means "process started" — users get errors during deploys.

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: shop
spec:
  selector: { app: api }
  ports:
    - port: 80          # service port
      targetPort: 8000  # container port
```

| Type | Reachable from |
|---|---|
| `ClusterIP` (default) | inside the cluster: `api.shop.svc.cluster.local`, or just `api` from the same namespace |
| `NodePort` | every node's IP on a port 30000–32767 |
| `LoadBalancer` | a cloud load balancer with a public or internal IP |
| `ExternalName` | DNS alias to an outside name |
| Headless (`clusterIP: None`) | DNS returns pod IPs directly (StatefulSets) |

Services balance only to pods that are **Ready**. Check targets:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=api
```

Empty endpoints = selector labels do not match the pods, or no pod is ready. This is the most common "service does not work" cause.

## Ingress

Routes HTTP(S) by host and path to Services. Needs an **ingress controller** installed (ingress-nginx, Traefik, a cloud controller). The newer **Gateway API** (`Gateway`, `HTTPRoute`) does the same with a cleaner model and is where the ecosystem is moving.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
  namespace: shop
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
    - hosts: [shop.example.com]
      secretName: shop-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api, port: { number: 80 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: web, port: { number: 80 } }
```

**cert-manager** obtains and renews Let's Encrypt certificates and stores them in the `shop-tls` Secret.

## ConfigMap and Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: shop
data:
  LOG_LEVEL: info
  FEATURE_X: "true"
---
apiVersion: v1
kind: Secret
metadata:
  name: api-db
  namespace: shop
type: Opaque
stringData:
  password: change-me
```

Facts that matter:
- Secrets are only **base64-encoded**, not encrypted. Enable encryption at rest for etcd, restrict RBAC on `secrets`, and do not commit Secret YAML to Git — use **External Secrets Operator** (syncs from Vault/cloud secret managers), **Sealed Secrets**, or SOPS (lesson 35).
- Env vars from ConfigMaps/Secrets are read **at pod start**. Changing them needs `kubectl rollout restart` (Helm/Kustomize can add a hash annotation to do this automatically).
- Mounted files update in the pod after a short delay (not with `subPath`).

## Storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads
  namespace: shop
spec:
  accessModes: [ReadWriteOnce]   # one node at a time; ReadWriteMany needs NFS/EFS-like storage
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

A **StorageClass** provisions disks dynamically (cloud disks, Longhorn, Ceph). For databases use a **StatefulSet** with `volumeClaimTemplates` — or, very often, a managed database outside the cluster.

## Namespaces, quotas, access

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: shop-quota, namespace: shop }
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.memory: 32Gi
    pods: "50"
```

- **RBAC**: `Role`/`ClusterRole` (what) + `RoleBinding` (who). Give each app its own `ServiceAccount` with minimal rights.
- **NetworkPolicy**: by default every pod can reach every pod. Add a default-deny policy per namespace and allow only needed flows (needs a CNI that enforces it, e.g. Calico or Cilium).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-from-api-only, namespace: shop }
spec:
  podSelector: { matchLabels: { app: db } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: api } }
      ports:
        - port: 5432
```

## Practice

1. Deploy a two-tier app (web + api, or the compose stack from lesson 43) into namespace `shop` with Deployments, Services, a ConfigMap and a Secret.
2. Install ingress-nginx in your local cluster and route `/api` and `/` by path.
3. Break the Service selector on purpose, find it via empty endpoints, fix it.
4. Change a ConfigMap value; observe the env var does not change until `rollout restart`.
5. Add a default-deny NetworkPolicy, then allow only web → api.
6. Run PostgreSQL as a StatefulSet with a PVC; delete the pod and check data survives.

## Check yourself

- Why does a rolling update without a readiness probe still cause errors?
- A Service exists but requests fail. What is the first thing you check?
- Are Kubernetes Secrets encrypted? What do you do about it?

## Next
[Lesson 52. Requests, limits, probes, autoscaling](52-resources-probes-autoscaling.md)
