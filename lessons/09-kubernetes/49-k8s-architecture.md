# Lesson 49. Kubernetes architecture and core objects

## Goal
Understand what Kubernetes does, how its parts fit together, and get a local cluster running.

## What problem it solves

With containers on many machines you need something to: place them on nodes with free resources, restart them when they crash, replace them when a node dies, roll out new versions without downtime, give them stable network names, and scale them. Kubernetes does this with a **declarative** model: you describe the desired state, controllers keep reality matching it.

## Architecture

```
            Control plane                                Worker nodes
┌──────────────────────────────────────┐      ┌───────────────────────────────┐
│ kube-apiserver  ← everything talks   │      │ kubelet  (runs pods via CRI)  │
│                   to the API         │◀────▶│ container runtime (containerd)│
│ etcd            ← cluster state (KV) │      │ kube-proxy / CNI dataplane    │
│ kube-scheduler  ← picks a node       │      │   pods pods pods              │
│ controller-mgr  ← reconcile loops    │      └───────────────────────────────┘
│ cloud-controller← LBs, volumes, nodes│
└──────────────────────────────────────┘
```

- **kube-apiserver**: the only component that talks to etcd. `kubectl`, controllers and kubelets all use its REST API.
- **etcd**: consistent key-value store with the whole cluster state. Back it up; lose it and you lose the cluster.
- **scheduler**: assigns new pods to nodes based on requests, affinity, taints.
- **controller manager**: loops like "Deployment wants 3 replicas, 2 exist → create 1".
- **kubelet**: agent on each node: starts containers, runs probes, reports status.
- **CNI plugin** (Calico, Cilium, Flannel...): pod networking: every pod gets its own IP, all pods can reach each other.
- **CoreDNS**: DNS names for services.

Managed Kubernetes (EKS, GKE, AKS) runs the control plane for you; you manage nodes and workloads.

## Core objects

| Object | What it is |
|---|---|
| **Pod** | smallest unit: one or more containers sharing network (same IP) and volumes. Disposable. |
| **ReplicaSet** | keeps N identical pods running (you rarely create it directly) |
| **Deployment** | manages ReplicaSets: rolling updates, rollbacks. For stateless apps |
| **StatefulSet** | pods with stable names (`db-0`, `db-1`) and their own volumes. For databases, queues |
| **DaemonSet** | one pod per node: log agents, node exporters, CNI |
| **Job / CronJob** | run to completion / on a schedule |
| **Service** | stable virtual IP + DNS name in front of a set of pods |
| **Ingress / Gateway** | HTTP(S) routing from outside to services |
| **ConfigMap / Secret** | configuration and sensitive data for pods |
| **PersistentVolumeClaim** | a request for storage, bound to a PersistentVolume |
| **Namespace** | a scope for names, quotas and access control |

Everything is selected by **labels**: a Service finds its pods by `app: api`, a Deployment owns pods with matching labels.

## Every object has the same shape

```yaml
apiVersion: apps/v1          # API group/version
kind: Deployment             # type
metadata:
  name: api
  namespace: shop
  labels: { app: api }
spec:                        # desired state - you write this
  replicas: 3
  # ...
status:                      # actual state - Kubernetes writes this
  readyReplicas: 3
```

## A local cluster

Choose one:

```bash
# kind: Kubernetes in Docker
kind create cluster --name lab
# minikube
minikube start --cpus 2 --memory 4g
# k3s: lightweight, real single-node cluster on a VM
curl -sfL https://get.k3s.io | sh -
```

Install `kubectl` (matching the cluster version ±1), then:

```bash
kubectl cluster-info
kubectl get nodes -o wide

# system pods: coredns, kube-proxy, CNI...
kubectl get pods -A
```

## Building a cluster with kubeadm (overview)

Good to do once to understand the parts:

1. On every node: disable swap (or configure kubelet for it), load `overlay` and `br_netfilter`, set `net.bridge.bridge-nf-call-iptables=1` and `net.ipv4.ip_forward=1` (lesson 36), install containerd, kubelet, kubeadm, kubectl.
2. `kubeadm init --pod-network-cidr=10.244.0.0/16` on the control plane.
3. Install a CNI plugin.
4. `kubeadm join ...` on workers.

## Your first workload

```bash
kubectl create deployment hello --image=nginx:1.27 --replicas=2
kubectl expose deployment hello --port=80
kubectl get deploy,rs,pods,svc -l app=hello

# watch them come back
kubectl delete pod -l app=hello --wait=false && kubectl get pods -w

# curl localhost:8080
kubectl port-forward svc/hello 8080:80

kubectl delete deployment hello && kubectl delete svc hello
```

You deleted pods, and the Deployment's controller created new ones. That is the reconcile loop in action.

## Practice

1. Create a kind or k3s cluster. List all system pods and explain what each one does.
2. Create a Deployment with 3 replicas, delete one pod, and watch it recreated.
3. `kubectl get deploy hello -o yaml`: find `spec` and `status`.
4. If you have 2–3 VMs: build a cluster with kubeadm following the official docs.

## Check yourself

- Which component stores cluster state, and which is the only one that talks to it?
- Pod vs Deployment: why do you almost never create bare pods?
- How does a Service know which pods to send traffic to?

---

<p align="left">
  <a href="media/49-k8s-architecture.mp4"><img width="640" src="media/49-k8s-architecture.gif" alt="Lesson 49 animation: Kubernetes architecture and core objects"></a>
</p>

## Next
[Lesson 50. kubectl in daily work](50-kubectl.md)
