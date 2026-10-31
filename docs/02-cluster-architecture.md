# 02 — Kubernetes Cluster Architecture and Control Plane

## 1. Objective

Create a single-node Kubernetes cluster using kind, identify the core system components, and perform basic administrative diagnostics.

**Related laboratories:** K8S-005 and K8S-006.

## 2. Cluster Creation

The Kubernetes cluster was created using:

```bash
kind create cluster --name k8s-lab --wait 120s
```

Observed result:

```text
Creating cluster "k8s-lab" ...
✓ Ensuring node image (kindest/node:v1.37.0)
✓ Preparing nodes
✓ Writing configuration
✓ Starting control-plane
✓ Installing CNI
✓ Installing StorageClass
✓ Waiting for control-plane = Ready

Ready after 18s
Set kubectl context to "kind-k8s-lab"
```

The cluster uses Kubernetes v1.37.0 and a single control-plane node.

## 3. Cluster Verification

Commands:

```bash
kind get clusters
kubectl cluster-info
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get namespaces
```

Observed configuration:

| Property | Value |
|---|---|
| Cluster name | k8s-lab |
| kubectl context | kind-k8s-lab |
| Kubernetes | v1.37.0 |
| Node | k8s-lab-control-plane |
| Node status | Ready |
| Node role | control-plane |
| Node operating system | Debian GNU/Linux 13 (trixie) |
| Architecture | ARM64 |
| Container runtime | containerd 2.3.4 |
| Node internal IP | 172.18.0.2 |
| Pod CIDR | 10.244.0.0/24 |
| Service CIDR | 10.96.0.0/16 |

The Kubernetes API Server was exposed to the macOS host through a local port, observed as:

```text
https://127.0.0.1:50731
```

This port is dynamically assigned and may change when the cluster is recreated.

## 4. Architecture Overview

```text
MacBook — macOS ARM64
|
+-- kubectl
    |
    +-- Kubernetes API (localhost)
        |
        +-- Colima Linux VM
            |
            +-- Docker Engine
                |
                +-- kind node container
                    |
                    +-- kubelet
                    +-- containerd
                    |
                    +-- Control Plane
                        +-- kube-apiserver
                        +-- etcd
                        +-- kube-scheduler
                        +-- kube-controller-manager
                    |
                    +-- Networking
                        +-- kindnet
                        +-- kube-proxy
                        +-- CoreDNS
```

The entire local Kubernetes node runs inside a Docker container managed by Colima.

Within that node, kubelet and containerd manage Kubernetes workloads.

## 5. Control Plane Components

### kube-apiserver

The API Server is the primary interface to Kubernetes.

It receives requests from kubectl and other clients, performs authentication and authorization, and handles Kubernetes API operations.

### etcd

etcd is a distributed key-value store used to maintain Kubernetes cluster state.

In this laboratory, etcd runs on the same control-plane node.

### kube-scheduler

The Scheduler assigns unscheduled Pods to suitable nodes based on scheduling constraints and available resources.

### kube-controller-manager

Runs controllers that continuously reconcile actual cluster state with the desired state.

Examples include node management and workload controllers.

### kubelet

The node agent responsible for ensuring that containers belonging to assigned Pods are running.

### containerd

The container runtime used by Kubernetes within the kind node.

It is separate from the Docker Engine running the outer kind node container.

## 6. System Pods

The following components were observed in the kube-system namespace:

| Component | Running Pods |
|---|---:|
| CoreDNS | 2 |
| etcd | 1 |
| kindnet | 1 |
| kube-apiserver | 1 |
| kube-controller-manager | 1 |
| kube-proxy | 1 |
| kube-scheduler | 1 |
| **Total** | **8** |

All eight system Pods were running with zero restarts during the initial verification.

A local-path storage provisioner was also observed in the separate `local-path-storage` namespace.

## 7. Node Health Diagnostics

Command:

```bash
kubectl describe node k8s-lab-control-plane
```

Observed node conditions:

| Condition | Status |
|---|---|
| Ready | True |
| MemoryPressure | False |
| DiskPressure | False |
| PIDPressure | False |
| Unschedulable | False |
| Taints | None |

No node-level resource pressure was reported during the inspection.

The single-node laboratory allows application Pods to be scheduled on the control-plane node.

## 8. Resource Capacity and Requests

The node reported:

```text
CPU capacity:       2
Memory capacity:    3998420Ki
Maximum Pods:       110
```

At the time of the initial inspection, the system workloads declared:

```text
CPU requests:       950m (47%)
Memory requests:    290Mi (7%)
Memory limits:      340Mi (8%)
```

### CPU units

Kubernetes CPU quantities can be expressed in millicores.

```text
1000m = 1 CPU
500m  = 0.5 CPU
250m  = 0.25 CPU
100m  = 0.1 CPU
```

CPU requests influence scheduling decisions.

They do not represent actual CPU utilization.

Memory requests are also scheduling reservations, not direct measurements of memory consumption.

## 9. Static Pods

The API Server configuration was inspected using:

```bash
kubectl get pod \
  kube-apiserver-k8s-lab-control-plane \
  -n kube-system \
  -o yaml
```

The Pod contained annotations indicating that its configuration originated from a local file:

```yaml
kubernetes.io/config.source: file
```

The API Server runs as a static Pod managed directly by kubelet.

Important configuration values included:

```text
--secure-port=6443
--authorization-mode=Node,RBAC
--etcd-servers=https://127.0.0.1:2379
--service-cluster-ip-range=10.96.0.0/16
```

The API Server uses `hostNetwork: true` and has critical system priority.

## 10. API Server Logs

Command:

```bash
kubectl logs \
  -n kube-system \
  kube-apiserver-k8s-lab-control-plane \
  --tail=20
```

The observed logs showed normal startup and configuration events, including initialization of Kubernetes system resources and Service IP allocation.

No evidence of an API Server failure was found in the inspected output.

## 11. API Server Readiness

Command:

```bash
kubectl get --raw='/readyz?verbose'
```

Selected results:

```text
[+]ping ok
[+]log ok
[+]etcd ok
[+]etcd-readiness ok
[+]informer-sync ok

readyz check passed
```

This confirmed that the API Server passed the reported readiness checks.

## 12. Kubernetes Networking

The laboratory uses multiple address spaces.

| Network | Purpose |
|---|---|
| 127.0.0.1 | Local API access from macOS |
| 172.18.0.0/16 | Docker network containing the kind node |
| 10.244.0.0/24 | Pod CIDR assigned to the node |
| 10.96.0.0/16 | Kubernetes Service address range |

The node's observed internal address was `172.18.0.2`.

CoreDNS was subsequently observed at the Service IP `10.96.0.10`.

Application Service and Pod networking are covered in the Services and DNS laboratory documentation.

## 13. Useful Administration Commands

List all Pods across namespaces:

```bash
kubectl get pods -A
```

Inspect nodes:

```bash
kubectl get nodes -o wide
kubectl describe node k8s-lab-control-plane
```

Inspect system components:

```bash
kubectl get pods -n kube-system
```

Check cluster health:

```bash
kubectl cluster-info
kubectl get --raw='/readyz?verbose'
```

Inspect the underlying kind container:

```bash
docker ps
```

## 14. Lessons Learned

1. Kubernetes uses a declarative control model.
2. The Control Plane manages and coordinates cluster operations.
3. kubelet runs on each node and manages assigned Pods.
4. Kubernetes in kind uses containerd inside node containers.
5. Static Pods allow essential Control Plane components to start without depending on the Kubernetes API.
6. Node conditions help identify resource pressure and readiness problems.
7. CPU and memory requests are not measurements of real utilization.
8. A single-node cluster is suitable for learning but does not provide node-level high availability.

## 15. Laboratory Result

**Status: Completed**

The kind cluster was successfully created and its fundamental Control Plane components were verified.

The API Server readiness checks passed, system Pods were running, and the node reported no resource pressure.
