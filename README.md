# Kubernetes Administration Lab

A hands-on Kubernetes learning project focused on Linux system administration, container orchestration, troubleshooting, networking, and DevOps automation.

The project uses a local Kubernetes cluster running on Apple Silicon, with future plans to integrate Ansible, Terraform, GitHub Actions, and Oracle Cloud Infrastructure.

## 1. Project Objectives

- Understand Kubernetes architecture and core components.
- Deploy and manage containerized applications.
- Practice declarative configuration using YAML manifests.
- Investigate cluster events, logs, and application failures.
- Learn self-healing, scaling, service discovery, and networking.
- Develop practical Kubernetes administration and DevOps skills.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| Host | Apple Silicon MacBook M2 |
| Host OS | macOS 26.7.1 (ARM64) |
| Virtualization | Colima 0.10.3 |
| VM resources | 2 vCPU, 4 GiB RAM, 30 GiB disk |
| Container Engine | Docker Engine 29.5.2 |
| Docker CLI | 29.9.2 |
| Kubernetes cluster | kind v0.33.0 |
| Kubernetes version | v1.37.0 |
| kubectl | v1.37.2 |
| Cluster name | k8s-lab |
| Node OS | Debian GNU/Linux 13 |
| Kubernetes container runtime | containerd 2.3.4 |

## 3. Architecture

```text
macOS (Apple Silicon ARM64)
|
+-- kubectl
|
+-- Colima (Linux VM)
    |
    +-- Docker Engine
        |
        +-- kind: k8s-lab
            |
            +-- Control Plane
                |
                +-- kube-apiserver
                +-- etcd
                +-- kube-scheduler
                +-- kube-controller-manager
                +-- kubelet
                +-- containerd
                +-- CoreDNS
                +-- kube-proxy
                +-- kindnet
                |
                +-- Namespace: lab-apps
                    |
                    +-- Deployment: nginx-deployment
                    |   +-- ReplicaSet
                    |       +-- 5 Nginx Pods
                    |
                    +-- Service: nginx-service
                    +-- Test Pod: dns-client
```

The cluster consists of a single control-plane node. It is suitable for local training but does not provide node-level high availability.

## 4. Installation and Cluster Setup

Install the necessary tools:

```bash
brew install colima docker kubectl kind
```

Start Colima:

```bash
colima start \
  --cpus 2 \
  --memory 4 \
  --disk 30 \
  --vm-type vz \
  --runtime docker
```

Verify Docker:

```bash
docker version
docker run --rm hello-world
```

Create the Kubernetes cluster:

```bash
kind create cluster --name k8s-lab --wait 120s
```

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

## 5. Deploying Nginx

Create the application namespace:

```bash
kubectl create namespace lab-apps
```

Apply the manifests:

```bash
kubectl apply -f manifests/nginx-deployment.yaml
kubectl apply -f manifests/nginx-service.yaml
```

Verify the deployment:

```bash
kubectl rollout status deployment/nginx-deployment -n lab-apps
kubectl get deployments,replicasets,pods -n lab-apps
kubectl get services -n lab-apps
```

The Deployment currently maintains five Nginx replicas.

## 6. Kubernetes Networking

The Nginx Service uses:

| Property | Value |
|---|---|
| Service name | nginx-service |
| Namespace | lab-apps |
| Service type | ClusterIP |
| Cluster IP observed | 10.96.201.49 |
| Port | 80/TCP |
| DNS name | nginx-service.lab-apps.svc.cluster.local |
| Backend selector | app=nginx |

The ClusterIP is assigned dynamically and may change if the Service is recreated.

Test the Service from a Pod:

```bash
kubectl run dns-client \
  -n lab-apps \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

```bash
kubectl exec -n lab-apps dns-client -- \
  nslookup nginx-service.lab-apps.svc.cluster.local
```

```bash
kubectl exec -n lab-apps dns-client -- \
  wget -S -O /dev/null http://nginx-service:80/
```

The internal HTTP test returned `HTTP/1.1 200 OK`.

## 7. Completed Laboratories

| Lab | Topic | Result |
|---|---|---|
| K8S-001 | macOS environment diagnostics | Completed |
| K8S-002 | Kubernetes tools installation | Completed |
| K8S-003 | Resource planning | Completed |
| K8S-004 | Colima and Docker configuration | Completed |
| K8S-005 | First Kubernetes cluster | Completed |
| K8S-006 | Control Plane inspection | Completed |
| K8S-007 | First Nginx Pod | Completed |
| K8S-008 | HTTP testing and port-forwarding | Completed |
| K8S-009 | Deployments and ReplicaSets | Completed |
| K8S-010 | Pod self-healing | Completed |
| K8S-011 | Horizontal scaling | Completed |
| K8S-012 | ClusterIP Services and EndpointSlices | Completed |
| K8S-013 | Kubernetes DNS and inter-Pod communication | Completed |
| K8S-014 | Service traffic distribution | Completed |
| K8S-015 | Project documentation and Git setup | In progress |

## 8. Documentation

Detailed technical documentation is organized into:

- [Environment Setup](docs/01-environment.md)
- [Cluster Architecture](docs/02-cluster-architecture.md)
- [Pods and Deployments](docs/03-pods-deployments.md)
- [Self-Healing and Scaling](docs/04-self-healing-scaling.md)
- [Services and DNS](docs/05-services-dns.md)

These documents will be completed as part of the project documentation checkpoint.

## 9. Troubleshooting Experience

The project includes practical investigation of:

- A missing Docker Desktop credential helper after Docker Desktop removal.
- Initial container image downloads and Pod startup transitions.
- HTTP 404 responses for missing Nginx favicon resources.
- BusyBox DNS search-path behavior and NXDOMAIN responses.
- ReplicaSet recovery after manual Pod deletion.
- Uneven distribution of HTTP connections across Nginx replicas.

## 10. Planned Laboratories

Future topics include:

- Rolling updates and rollbacks.
- Readiness, liveness, and startup probes.
- ConfigMaps and Secrets.
- PersistentVolumes and PersistentVolumeClaims.
- RBAC and service accounts.
- Resource quotas and monitoring.
- Helm.
- Kubernetes automation with Ansible and Terraform.
- CI/CD with GitHub Actions.
- Deployment of the Java-based JobTracker application.

## 11. Security Notes

This is a local educational environment.

Do not commit kubeconfig files, private keys, API credentials, passwords, or sensitive Kubernetes Secret manifests to the repository.

Production deployments require additional security controls, resource management, monitoring, backups, and high-availability considerations.

## License

Educational project. License not yet selected.
