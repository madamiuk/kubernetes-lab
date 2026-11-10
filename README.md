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
| K8S-015 | Project documentation and Git setup | Completed |
| K8S-016 | Rolling Updates and Rollbacks | Completed |
| K8S-017 | Kubernetes Health Probes | Completed |
| K8S-018 | ConfigMaps and Secrets | Completed |
| K8S-019 | PersistentVolumes and PersistentVolumeClaims | Completed |
| K8S-020 | RBAC and Service Accounts | Completed |
| K8S-021 | Resource Requests, Limits and ResourceQuotas | Completed |
| K8S-022 | Metrics Server, Resource Metrics and CPU Throttling | Completed |
| K8S-023 | Horizontal Pod Autoscaler (HPA) | Completed |
| K8S-024 | Helm: Kubernetes Application Lifecycle Management | Completed |
| K8S-025 | Prometheus and Grafana: Kubernetes Monitoring and Dashboards | Completed |
| K8S-026 | Prometheus Alerting, Alertmanager and Kubernetes Health Monitoring | Completed |
| K8S-027 | Kubernetes Automation with Ansible: Declarative Application Management | Completed |

## 8. Documentation

Detailed technical documentation is organized into:

- [Environment Setup](docs/01-environment.md)
- [Cluster Architecture](docs/02-cluster-architecture.md)
- [Pods and Deployments](docs/03-pods-deployments.md)
- [Self-Healing and Scaling](docs/04-self-healing-scaling.md)
- [Services and DNS](docs/05-services-dns.md)
- [Rolling Updates and Rollbacks](docs/06-rolling-updates-rollbacks.md)
- [Health Probes](docs/07-health-probes.md)
- [ConfigMaps and Secrets](docs/08-configmaps-secrets.md)
- [PersistentVolumes and PersistentVolumeClaims](docs/09-persistent-volumes.md)
- [RBAC and Service Accounts](docs/10-rbac-service-accounts.md)
- [Resource Requests, Limits and ResourceQuotas](docs/11-resource-management.md)
- [Metrics Server, Resource Metrics and CPU Throttling](docs/12-metrics-monitoring.md)
- [Horizontal Pod Autoscaler (HPA)](docs/13-horizontal-pod-autoscaler.md)
- [Helm: Kubernetes Application Lifecycle Management](docs/14-helm.md)
- [Prometheus and Grafana: Kubernetes Monitoring and Dashboards](docs/15-prometheus-grafana.md)
- [Prometheus Alerting and Alertmanager: Kubernetes Health Monitoring](docs/16-prometheus-alertmanager.md)
- [Kubernetes Automation with Ansible: Declarative Application Management](docs/17-ansible-kubernetes.md)

The documentation covers completed laboratories K8S-001 through K8S-027, including cluster administration, application deployments, networking, rolling updates, rollbacks, health probes, ConfigMaps, Secrets, persistent storage, Kubernetes RBAC, resource management, Metrics Server, CPU throttling, Horizontal Pod Autoscaler (HPA), Helm-based application lifecycle management, Kubernetes monitoring with Prometheus and Grafana, and Prometheus alerting with Alertmanager.

The Helm laboratory includes a custom chart located in `charts/nginx-lab/`, covering chart configuration, template rendering, Kubernetes API validation, release installation, upgrades, rollbacks, test hooks, chart versioning, and rolling updates.
The monitoring laboratory deploys Prometheus and Grafana using version-pinned Helm charts, resource requests and limits, persistent storage, Kubernetes Secrets, and declarative dashboard provisioning. It includes PromQL queries, metrics collection from Kubernetes and exporters, troubleshooting on ARM64, and recovery tests following Pod restarts.

The alerting laboratory extends the monitoring stack with Alertmanager, Prometheus alert rules, alert routing, and Silence management. It includes controlled alert generation, verification of pending and firing states, Kubernetes Pod readiness failure simulation, and automatic alert resolution. The final configuration retains reusable Pod readiness and node MemoryPressure alert rules.

The Ansible automation laboratory introduces declarative Kubernetes resource management using the kubernetes.core collection. It covers cluster inspection, Namespace, Deployment and Service provisioning, replica scaling, idempotency, check mode, automated Deployment readiness verification, shared configuration variables, and fail-fast validation of deployment parameters. The complete application deployment is managed through a single Ansible playbook.

Monitoring configurations and the Kubernetes overview dashboard are maintained in `monitoring/`.

## 9. Troubleshooting Experience

The project includes practical investigation of:

- A missing Docker Desktop credential helper after Docker Desktop removal.
- Initial container image downloads and Pod startup transitions.
- HTTP 404 responses for missing Nginx favicon resources.
- BusyBox DNS search-path behavior and NXDOMAIN responses.
- ReplicaSet recovery after manual Pod deletion.
- Uneven distribution of HTTP connections across Nginx replicas.
- Helm `values.yaml` indentation errors and unexpected default image tags.
- Helm chart metadata changes triggering Kubernetes Deployment rolling updates.
- Prometheus Config Reloader `ImagePullBackOff` caused by an image without ARM64 support, resolved by pinning a compatible image version through Helm.
- Grafana Helm installation timeout caused by an initial container image download exceeding the configured timeout.
- Grafana web login failures investigated using API authentication tests and application logs.
- Grafana dashboard persistence and declarative recovery verified after a Deployment restart.

## 10. Planned Laboratories

Future topics include:

- External Alertmanager notification channels and end-to-end notification delivery testing.
- Kubernetes Helm release automation with Ansible.
- Kubernetes infrastructure provisioning with Terraform.
- CI/CD with GitHub Actions.
- Deployment of the Java-based JobTracker application.

## 11. Security Notes

This is a local educational environment.

Do not commit kubeconfig files, private keys, API credentials, passwords, or sensitive Kubernetes Secret manifests to the repository.

Production deployments require additional security controls, resource management, monitoring, backups, and high-availability considerations.

## License

Educational project. License not yet selected.
