# Lesson 53. Helm and GitOps

## Goal
Package Kubernetes apps for many environments and deploy them from Git automatically and auditably.

## The problem

The same app in dev, staging and prod differs in image tag, replicas, resources, hostnames and secrets. Copy-pasting YAML per environment drifts quickly. Two common answers:

- **Kustomize**: plain YAML base + patches per environment. Built into `kubectl -k`.
- **Helm**: templated charts with values files, plus release management (install, upgrade, rollback).

## Kustomize

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── staging/kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: shop-prod
resources:
  - ../../base
images:
  - name: ghcr.io/example/api
    newTag: "1.4.2"
patches:
  - path: replicas-patch.yaml
configMapGenerator:
  - name: api-config
    literals: [LOG_LEVEL=info]     # generated name has a hash → pods restart on change
```

```bash
kubectl kustomize overlays/prod
kubectl apply -k overlays/prod
```

- `kubectl kustomize overlays/prod`: render

## Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo ingress-nginx
helm show values ingress-nginx/ingress-nginx > values.yaml
helm install ingress ingress-nginx/ingress-nginx -n ingress --create-namespace -f values.yaml
helm list -A
helm upgrade ingress ingress-nginx/ingress-nginx -n ingress -f values.yaml --version 4.11.3
helm history ingress -n ingress
helm rollback ingress 1 -n ingress
helm uninstall ingress -n ingress
helm template ./chart -f values-prod.yaml
helm diff upgrade api ./chart -f values-prod.yaml
```

- `helm template ./chart -f values-prod.yaml`: render locally, review the YAML
- `helm diff upgrade api ./chart -f values-prod.yaml`: plugin: what will change

Always pin chart versions (`--version`).

### Your own chart

```bash
helm create api
```

- `helm create api`: scaffold

```
api/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── _helpers.tpl
```

- `├── Chart.yaml`: name, version (of the chart), appVersion
- `├── values.yaml`: defaults

`values.yaml`:

```yaml
image:
  repository: ghcr.io/example/api
  tag: "1.4.2"
replicaCount: 3
resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits: { memory: 512Mi }
ingress:
  enabled: true
  host: shop.example.com
```

`templates/deployment.yaml` (excerpt):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "api.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

The `checksum/config` annotation changes when the ConfigMap changes, so pods roll automatically.

```bash
helm lint ./api
helm upgrade --install api ./api -n shop -f values-prod.yaml --atomic --timeout 5m
```

`--atomic` rolls back automatically if the release does not become healthy.

## GitOps

**Git is the single source of truth for what runs in the cluster.** An agent in the cluster continuously compares the desired state in Git with the live state and fixes differences.

```
developer → PR to app repo → CI builds image ghcr.io/example/api:sha-3f2c1ab
                                     │
                     CI (or a bot) opens PR to config repo: tag → sha-3f2c1ab
                                     │
                          review + merge to main
                                     │
          Argo CD / Flux in the cluster sees the change → applies → reports health
```

Benefits:
- No cluster credentials in CI (pull model).
- Every change is a reviewed commit: audit log and easy rollback (`git revert`).
- **Drift** is detected and reverted: manual `kubectl edit` changes are undone (or flagged).
- Rebuilding a cluster = point the agent at the repo.

### Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8443:443
```

`Application`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/k8s-config.git
    targetRevision: main
    path: apps/api/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: shop-prod
  syncPolicy:
    automated:
      prune: true        # delete objects removed from Git
      selfHeal: true     # revert manual changes
    syncOptions: [CreateNamespace=true]
```

Patterns: "app of apps" / `ApplicationSet` to manage many apps and clusters; Argo Rollouts for canary and blue-green with automatic analysis of metrics (lesson 44).

**Flux** is the other popular option: same ideas, controllers (`GitRepository`, `Kustomization`, `HelmRelease`) and no UI by default.

### Repository layout

Keep application code and deployment config in **separate repos** (or at least separate folders with separate review rules):

```
k8s-config/
├── apps/
│   ├── api/{base,overlays/{staging,prod}}
│   └── web/...
├── platform/
└── clusters/
    ├── staging/
    └── prod/
```

- `├── platform/`: ingress, cert-manager, monitoring, external-secrets
- `├── staging/`: which apps/versions run here

Secrets do not go into this repo in plain form: use External Secrets, Sealed Secrets or SOPS.

## Practice

1. Convert your app from lesson 51 into a Kustomize base with staging and prod overlays.
2. Create a Helm chart for it; render with `helm template`, install with `--atomic`, upgrade, roll back.
3. Install Argo CD in a local cluster and point an Application to a Git repo of yours.
4. Change the image tag in Git and watch Argo CD sync. Then `kubectl scale` by hand and watch `selfHeal` revert it.
5. Roll back by reverting the commit.

## Check yourself

- Helm vs Kustomize: what does each one do best?
- Why is the pull model of GitOps more secure than CI running `kubectl apply`?
- What happens to a manual `kubectl edit` with `selfHeal: true`?

## Next
[Lesson 54. Kubernetes troubleshooting](54-k8s-troubleshooting.md)
