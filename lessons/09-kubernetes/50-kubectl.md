# Lesson 50. kubectl in daily work

## Goal
Navigate any cluster fast: find resources, read their state, get logs, get a shell, and apply changes from YAML.

## Contexts and namespaces

```bash
kubectl config get-contexts
kubectl config use-context prod-eu       # switch cluster - check it before every destructive command
kubectl config set-context --current --namespace=shop
kubectl get ns
```

Tools that help: `kubectx` / `kubens`, a shell prompt that shows the current context, `k9s` (terminal UI).

```bash
alias k=kubectl
source <(kubectl completion bash); complete -o default -F __start_kubectl k
```

## Get and describe

```bash
kubectl get pods                         # current namespace
kubectl get pods -A                      # all namespaces
kubectl get pods -o wide                 # node, pod IP
kubectl get pods -l app=api              # by label
kubectl get pods --field-selector=status.phase!=Running
kubectl get pods --sort-by=.status.containerStatuses[0].restartCount
kubectl get deploy,svc,ingress,pvc
kubectl get all                          # not really all, but the common ones
kubectl get pod api-7d9c -o yaml
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl describe pod api-7d9c            # events at the bottom - read them first!
kubectl get events --sort-by=.lastTimestamp
kubectl explain deployment.spec.strategy # built-in docs for any field
kubectl api-resources                    # all object types and short names
```

## Logs and shells

```bash
kubectl logs api-7d9c
kubectl logs api-7d9c -c sidecar         # specific container
kubectl logs api-7d9c --previous         # the crashed instance before the restart
kubectl logs -f deploy/api               # one pod of the deployment
kubectl logs -l app=api --tail=50 --prefix   # many pods
kubectl exec -it api-7d9c -- sh
kubectl exec api-7d9c -- env
kubectl debug -it api-7d9c --image=nicolaka/netshoot --target=api   # ephemeral debug container
kubectl debug node/worker-1 -it --image=ubuntu                       # shell on a node
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh  # throwaway pod for tests
kubectl port-forward svc/api 8080:80
kubectl cp api-7d9c:/tmp/dump.txt ./dump.txt
```

## Resource usage

```bash
kubectl top nodes                        # needs metrics-server
kubectl top pods -A --sort-by=memory
kubectl describe node worker-1           # allocated requests/limits, conditions, taints
```

## Declarative workflow

Keep YAML in Git and apply it. Avoid editing live objects by hand in production.

```bash
kubectl apply -f k8s/                    # create or update
kubectl diff -f k8s/                     # what would change
kubectl apply -f k8s/ --dry-run=server   # validate against the API
kubectl delete -f k8s/
kubectl apply -k overlays/prod           # kustomize
```

Generate YAML instead of writing it from scratch:

```bash
kubectl create deployment api --image=ghcr.io/org/api:1.4.2 --replicas=3 --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip api --tcp=80:8000 --dry-run=client -o yaml > svc.yaml
kubectl create configmap app-config --from-file=config.yaml --dry-run=client -o yaml
kubectl create secret generic db --from-literal=password=change-me --dry-run=client -o yaml
```

## Imperative changes (ok for labs and emergencies)

```bash
kubectl scale deploy/api --replicas=5
kubectl set image deploy/api api=ghcr.io/org/api:1.4.3
kubectl rollout status deploy/api
kubectl rollout history deploy/api
kubectl rollout undo deploy/api                    # back to previous version
kubectl rollout restart deploy/api                 # restart all pods, e.g. after config change
kubectl label pod api-7d9c debug=true
kubectl annotate deploy api owner=team-shop
kubectl cordon worker-1                            # no new pods on this node
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data   # move pods away for maintenance
kubectl uncordon worker-1
```

After an emergency imperative change, bring Git back in sync — otherwise the next `apply` or GitOps sync reverts it.

## Permissions

```bash
kubectl auth can-i create deployments -n shop
kubectl auth can-i '*' '*' --as=system:serviceaccount:shop:api
kubectl auth whoami
```

## Practice

1. Set up aliases and completion. Install `k9s` and explore your cluster with it.
2. Generate Deployment and Service YAML with `--dry-run=client -o yaml`, store them in a folder, `kubectl apply -f`.
3. Change the image tag in the file, run `kubectl diff`, then apply and watch `rollout status`.
4. Roll back with `rollout undo`. Check `rollout history`.
5. Use `kubectl debug` with a netshoot image to run `curl` and `dig` from inside the cluster.
6. Drain a node (in a multi-node kind cluster) and watch pods move.

## Check yourself

- Where do you look first when a pod is not healthy?
- How do you see logs of a container that already crashed and restarted?
- Why prefer `kubectl apply -f` from Git over `kubectl edit` in production?

## Next
[Lesson 51. Deployments, Services, Ingress, ConfigMaps, Secrets](51-workloads-networking-config.md)
