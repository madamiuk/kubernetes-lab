
# 15 — Prometheus and Grafana: Kubernetes Monitoring and Dashboard Provisioning

**Laboratory:** K8S-025
**Environment:** kind-k8s-lab
**Kubernetes:** v1.37.0
**Helm:** v4.3.1
**Prometheus:** v3.16.0
**Prometheus Helm chart:** 29.45.1
**Grafana:** v13.3.0
**Grafana Helm chart:** 14.4.0
**Status:** Practical experiments completed

## 1. Objective

This laboratory implemented a lightweight, reproducible monitoring stack for the local, single-node Kubernetes cluster. The objectives were to:

- Deploy Prometheus and Grafana using version-pinned Helm charts and repository-managed values files.
- Configure explicit CPU and memory requests and limits for all Prometheus containers and Grafana.
- Persist Prometheus time-series data and Grafana application data using local-path PersistentVolumeClaims.
- Collect Kubernetes API, Kubelet, cAdvisor, CoreDNS, node-exporter, kube-state-metrics, and Prometheus metrics.
- Verify scrape targets and execute PromQL queries using the Prometheus API and Grafana Explore.
- Reference administrator credentials from a pre-existing Kubernetes Secret without committing passwords.
- Provision a Grafana Prometheus datasource and a five-panel dashboard declaratively.
- Diagnose real-world image architecture, Helm timeout, and authentication problems.
- Verify that Grafana's provisioned dashboard survives a Deployment restart.

## 2. Architecture and Resource Budget

The stack runs in namespace `monitoring` on the single kind control-plane node:

```text
Kubernetes API / Kubelet / cAdvisor / CoreDNS
                   |              \
       node-exporter / kube-state-metrics
                   |
             Prometheus
        (PVC: 2Gi, 3d retention)
                   |
     Prometheus ClusterIP Service
                   |
                Grafana
       (PVC: 1Gi, provisioned dashboard)
                   |
          kubectl port-forward
                   |
          MacBook web browser
```

**Prometheus component requests and limits:**

| Container | CPU request | Memory request | CPU limit | Memory limit |
| --- | ---: | ---: | ---: | ---: |
| Prometheus server | 150m | 256Mi | 400m | 768Mi |
| ConfigMap reloader | 10m | 32Mi | 50m | 64Mi |
| kube-state-metrics | 30m | 64Mi | 100m | 128Mi |
| node-exporter | 20m | 32Mi | 100m | 64Mi |
| **Total** | **210m** | **384Mi** | **650m** | **1024Mi** |

Grafana is configured with requests of **100m CPU / 128Mi RAM** and limits of **300m CPU / 512Mi RAM**.

The node had **1710m (85%) CPU requests** after Prometheus deployment and **1810m (90%)** after Grafana was scheduled. These numbers are scheduler reservations, not measured CPU consumption. Node CPU limits totaled 2900m (145%) after Grafana, which is permitted because limits may be overcommitted. The lab's single-node configuration should not be interpreted as production capacity planning.

## 3. Repository Layout and Helm Repositories

Project files:

```text
monitoring/
├── prometheus-values.yaml
├── grafana-values.yaml
└── dashboards/
    └── kubernetes-overview.json

docs/
└── 15-prometheus-grafana.md
```

Add the chart repositories and update the local index:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update

helm search repo prometheus-community/prometheus --versions
helm search repo grafana-community/grafana --versions
```

The selected chart versions were **29.45.1** for Prometheus and **14.4.0** for Grafana. Save default chart values for investigation:

```bash
helm show values prometheus-community/prometheus --version 29.45.1 \
  > /tmp/k8s-025-prometheus-defaults.yaml
helm show values grafana-community/grafana --version 14.4.0 \
  > /tmp/k8s-025-grafana-defaults.yaml
```

Download charts locally to make linting and inspection reproducible:

```bash
mkdir -p /tmp/k8s-025-charts
helm pull prometheus-community/prometheus --version 29.45.1 \
  --untar --untardir /tmp/k8s-025-charts
helm pull grafana-community/grafana --version 14.4.0 \
  --untar --untardir /tmp/k8s-025-charts
```

**Helm 4 note:** `helm lint` did not support the `--version` flag in this lab; linting the downloaded chart path worked:

```bash
helm lint /tmp/k8s-025-charts/prometheus -f monitoring/prometheus-values.yaml
```

## 4. Prometheus Configuration

The repository file `monitoring/prometheus-values.yaml` specifies:

- One Prometheus server replica, three-day retention, and a 2Gi PVC using StorageClass `standard`.
- Requests and limits for the Prometheus server, ConfigMap reloader, kube-state-metrics, and node-exporter.
- kube-state-metrics and node-exporter enabled.
- Alertmanager and Pushgateway disabled to conserve laboratory resources.
- The ARM64-capable ConfigMap reloader image `quay.io/prometheus-operator/prometheus-config-reloader:v0.94.2` (reason described below).

Key configuration excerpt (refer to the complete repository values file for all parameters):

```yaml
server:
  replicaCount: 1
  retention: "3d"
  resources:
    requests: {cpu: "150m", memory: "256Mi"}
    limits: {cpu: "400m", memory: "768Mi"}
  persistentVolume:
    enabled: true
    storageClass: standard
    size: 2Gi

configmapReload:
  prometheus:
    image:
      repository: quay.io/prometheus-operator/prometheus-config-reloader
      tag: "v0.94.2"
      pullPolicy: IfNotPresent
    resources:
      requests: {cpu: "10m", memory: "32Mi"}
      limits: {cpu: "50m", memory: "64Mi"}

alertmanager:
  enabled: false
kube-state-metrics:
  enabled: true
  resources:
    requests: {cpu: "30m", memory: "64Mi"}
    limits: {cpu: "100m", memory: "128Mi"}
prometheus-node-exporter:
  enabled: true
  resources:
    requests: {cpu: "20m", memory: "32Mi"}
    limits: {cpu: "100m", memory: "64Mi"}
prometheus-pushgateway:
  enabled: false
```

Create the namespace idempotently:

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
```

Lint, render, and validate the chart against the Kubernetes API:

```bash
helm lint /tmp/k8s-025-charts/prometheus -f monitoring/prometheus-values.yaml
helm template prometheus /tmp/k8s-025-charts/prometheus \
  --namespace monitoring -f monitoring/prometheus-values.yaml \
  > /tmp/k8s-025-prometheus-rendered.yaml
kubectl apply --dry-run=server -n monitoring \
  -f /tmp/k8s-025-prometheus-rendered.yaml
```

All **15 rendered Kubernetes resources** passed server-side dry-run validation. Initial installation was attempted with:

```bash
helm install prometheus prometheus-community/prometheus \
  --version 29.45.1 --namespace monitoring \
  -f monitoring/prometheus-values.yaml --wait --timeout 5m
```

The initial installation failed because the server Pod's ConfigMap reloader image was unavailable for ARM64. After the fix in section 5, a successful upgrade created revision **2**:

```bash
helm upgrade prometheus prometheus-community/prometheus \
  --version 29.45.1 --namespace monitoring \
  -f monitoring/prometheus-values.yaml --wait --timeout 5m
```

## 5. Troubleshooting: Prometheus ImagePullBackOff on ARM64

**Symptoms:** Helm timed out waiting for `Deployment/monitoring/prometheus-server` to become available. The Pod was `1/2` ready with `ImagePullBackOff`. The other workloads were healthy, and the Prometheus PVC was already `Bound`.

Inspect the failed release, Pod, and events:

```bash
helm status prometheus -n monitoring
kubectl get pods -n monitoring -o wide
kubectl describe deployment prometheus-server -n monitoring
kubectl get events -n monitoring --sort-by=.lastTimestamp | tail -n 35
```

The key event was:

```text
Failed to pull image "quay.io/prometheus-operator/prometheus-config-reloader:v0.95.1":
no match for platform in manifest: not found
```

The node runs `linux/arm64`; the image manifest for `v0.95.1` exposed only `linux/amd64` entries. Verify the manifest lists on the Mac:

```bash
docker manifest inspect quay.io/prometheus-operator/prometheus-config-reloader:v0.95.1 \
  | jq -r '.manifests[]? | "\(.platform.os)/\(.platform.architecture)"'

docker manifest inspect quay.io/prometheus-operator/prometheus-config-reloader:v0.94.2 \
  | jq -r '.manifests[]? | "\(.platform.os)/\(.platform.architecture)"'
```

The `v0.94.2` manifest included `linux/arm64`, so the chart image tag was explicitly overridden in `configmapReload.prometheus.image.tag`. A new render confirmed the correct image before `helm upgrade`.

**Result:** Prometheus server reached `2/2 Running`, with the reloader on `v0.94.2`; kube-state-metrics and node-exporter remained `1/1 Running`. The existing 2Gi PVC and Helm release history were preserved.

**Operational lesson:** Check both the container image manifest's supported platforms and the node's architecture before changing CPU or memory settings. Pinning an older image is a validated *lab workaround*, not a blanket production recommendation; image support and security updates must be reassessed before production use.

## 6. Verifying Prometheus and PromQL

Forward the Prometheus ClusterIP service to a loopback-only port:

```bash
kubectl port-forward --address 127.0.0.1 -n monitoring \
  service/prometheus-server 19090:80
```

In a second terminal, verify health and readiness:

```bash
curl -fsS http://127.0.0.1:19090/-/healthy
curl -fsS http://127.0.0.1:19090/-/ready
```

The recorded responses were `Prometheus Server is Healthy.` and `Prometheus Server is Ready.`

Inspect active scrape targets:

```bash
curl -fsS http://127.0.0.1:19090/api/v1/targets \
  | jq -r '.data.activeTargets[] | [.labels.job, .health, .scrapeUrl] | @tsv'
```

**Observed result:** All **9/9 targets** were `up`, including Kubernetes API, Kubelet, cAdvisor, two CoreDNS endpoints, node-exporter, two kube-state-metrics endpoints, and Prometheus self-monitoring.

Run an instant PromQL query:

```bash
curl -fsSG --data-urlencode 'query=up' \
  http://127.0.0.1:19090/api/v1/query \
  | jq -r '.data.result[] | [.metric.job, .metric.instance, .value[1]] | @tsv'
```

Every observed `up` series had value `1`. In Grafana Explore, the query `sum(up)` returned `9`; selecting **Range** instead of **Instant** displayed the time-series line.

Other working Explore queries:

```promql
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m]))
node_memory_MemAvailable_bytes / 1024 / 1024
count(kube_pod_status_phase{phase="Running"} == 1)
```

## 7. Grafana Configuration and Credentials

The file `monitoring/grafana-values.yaml` configures:

- Single Grafana replica, image `docker.io/grafana/grafana:13.3.0-distroless` (manifest confirmed `linux/arm64`).
- CPU request/limit **100m/300m** and memory request/limit **128Mi/512Mi**.
- Persistent 1Gi PVC in StorageClass `standard`.
- Existing administrator Secret `grafana-admin` (keys `admin-user` and `admin-password`).
- Read-only default Prometheus datasource with stable UID `prometheus` and URL `http://prometheus-server.monitoring.svc.cluster.local`.
- A file-based dashboard provider named `kubernetes-lab`, targeting Grafana folder `Kubernetes Lab`.

The Secret was created directly in Kubernetes, not committed to Git:

```bash
read -r -s -p 'Grafana admin password: ' GRAFANA_PASSWORD
echo
kubectl create secret generic grafana-admin -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$GRAFANA_PASSWORD"
unset GRAFANA_PASSWORD

kubectl get secret grafana-admin -n monitoring -o json | jq -r '.data | keys[]'
```

**Observed result:** `admin-user` and `admin-password` were present. Avoid printing, storing in documentation, or committing Secret values. Kubernetes Secret data is not automatically encrypted at rest; production environments need stronger secret-management safeguards.

Selected values excerpt:

```yaml
replicas: 1
image:
  registry: docker.io
  repository: grafana/grafana
  tag: "13.3.0-distroless"
  pullPolicy: IfNotPresent
admin:
  existingSecret: grafana-admin
  userKey: admin-user
  passwordKey: admin-password
resources:
  requests: {cpu: "100m", memory: "128Mi"}
  limits: {cpu: "300m", memory: "512Mi"}
persistence:
  type: pvc
  enabled: true
  storageClassName: standard
  accessModes: [ReadWriteOnce]
  size: 1Gi
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        uid: prometheus
        type: prometheus
        access: proxy
        url: http://prometheus-server.monitoring.svc.cluster.local
        isDefault: true
        editable: false
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: kubernetes-lab
        orgId: 1
        folder: Kubernetes Lab
        type: file
        disableDeletion: false
        editable: false
        options:
          path: /var/lib/grafana/dashboards/kubernetes-lab
```

## 8. Validating and Installing Grafana

Lint and render Grafana:

```bash
helm lint /tmp/k8s-025-charts/grafana -f monitoring/grafana-values.yaml
helm template grafana /tmp/k8s-025-charts/grafana \
  --namespace monitoring -f monitoring/grafana-values.yaml \
  > /tmp/k8s-025-grafana-rendered.yaml
```

A server-side dry-run of the full rendered file failed on the test Pod `grafana-test` because its ServiceAccount had not been persisted during dry-run. It did **not** indicate failure of the Grafana Deployment. Rendering without Helm hooks avoided the dependency:

```bash
helm template grafana /tmp/k8s-025-charts/grafana \
  --namespace monitoring -f monitoring/grafana-values.yaml \
  --no-hooks > /tmp/k8s-025-grafana-no-hooks.yaml
kubectl apply --dry-run=server -n monitoring \
  -f /tmp/k8s-025-grafana-no-hooks.yaml
```

**Observed result:** All seven core Kubernetes resources passed server-side dry-run.

Initial deployment command:

```bash
helm install grafana grafana-community/grafana \
  --version 14.4.0 --namespace monitoring \
  -f monitoring/grafana-values.yaml --wait --timeout 5m
```

The release reached `failed`, revision **1**, because the Deployment was not ready within five minutes. This was not caused by ARM64 incompatibility.

## 9. Troubleshooting: Grafana Image Download Timeout

Inspect Pod status, init containers, events, and application logs:

```bash
kubectl describe pod -n monitoring -l app.kubernetes.io/name=grafana
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -o json \
  | jq -r '.items[].status.initContainerStatuses[]? | [.name, (.ready|tostring), (.state|tostring)] | @tsv'
kubectl get events -n monitoring --sort-by=.lastTimestamp
kubectl logs -n monitoring deployment/grafana --all-containers=true --tail=50
```

**Observed facts:** `init-chown-data` completed with exit code 0. Pulling the `13.3.0-distroless` image (~454 MB) took **7m47.968s**, exceeding Helm's five-minute timeout. Grafana subsequently started, became ready, and had zero container restarts. Brief readiness-probe `connection refused` messages appeared during startup.

Recover the existing release without deleting the PVC or installing a second release:

```bash
helm upgrade grafana grafana-community/grafana \
  --version 14.4.0 --namespace monitoring \
  -f monitoring/grafana-values.yaml --wait --timeout 10m
```

**Observed result:** Grafana reached `deployed`, revision **2**. Increasing a timeout is appropriate only when diagnostics demonstrate that startup or image transfer is progressing, not as a substitute for investigating errors.

## 10. Verifying Grafana API, Datasource, and Authentication

Port-forward Grafana on the host loopback interface:

```bash
kubectl port-forward --address 127.0.0.1 -n monitoring service/grafana 13000:80
```

The Grafana API confirmed a healthy SQLite database and version `13.3.0`:

```bash
curl -fsS http://127.0.0.1:13000/api/health | jq .
```

Verify the provisioned Prometheus datasource without displaying the administrator password:

```bash
read -r -s -p 'Grafana admin password: ' GRAFANA_PASSWORD
echo
curl -fsS -u "admin:${GRAFANA_PASSWORD}" \
  http://127.0.0.1:13000/api/datasources/uid/prometheus \
  | jq '{name, uid, type, url, isDefault, readOnly}'
unset GRAFANA_PASSWORD
```

**Observed result:** `name=Prometheus`, `uid=prometheus`, `isDefault=true`, and `readOnly=true`.

**Authentication troubleshooting:** At first, the web login form returned HTTP 401 with `password-auth.invalid`, although the authenticated API request to `/api/user` returned HTTP 200 and identified user ID `1`, login `admin`, with `isGrafanaAdmin=true`. The administrator password was changed and the web login subsequently succeeded. No evidence established that the original password's 32-character length caused the problem. Avoid treating an unproven browser input/encoding hypothesis as a confirmed Grafana defect.

## 11. Declarative Kubernetes Dashboard

The repository dashboard `monitoring/dashboards/kubernetes-overview.json` has UID `k8s-lab-overview` and five panels:

| Panel | Visualization | Query |
| --- | --- | --- |
| Healthy Targets | Stat | `sum(up)` |
| Running Pods | Stat | `count(kube_pod_status_phase{phase="Running"} == 1)` |
| Target Availability | Time series | `up` |
| Node CPU Usage | Time series | `100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))` |
| Node Memory Usage | Time series | `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` |

The node usage queries assume the lab's single monitored node. The Healthy Targets panel originally set its green threshold to **9**, matching the observed target count; this threshold is a fixed lab value and should be revised for a changing target inventory.

The `dashboardProviders` entry in `grafana-values.yaml` specifies the provider, folder, and path. Supply the standalone JSON file using `--set-file`:

```bash
jq empty monitoring/dashboards/kubernetes-overview.json
helm lint /tmp/k8s-025-charts/grafana \
  -f monitoring/grafana-values.yaml \
  --set-file dashboards.kubernetes-lab.kubernetes-overview.json=monitoring/dashboards/kubernetes-overview.json
helm template grafana /tmp/k8s-025-charts/grafana \
  --namespace monitoring -f monitoring/grafana-values.yaml \
  --set-file dashboards.kubernetes-lab.kubernetes-overview.json=monitoring/dashboards/kubernetes-overview.json \
  > /tmp/k8s-025-grafana-dashboard-rendered.yaml
```

The rendered Deployment mounted the dashboard at `/var/lib/grafana/dashboards/kubernetes-lab/kubernetes-overview.json` via ConfigMap.

Deploy it using the **same** `--set-file` parameter:

```bash
helm upgrade grafana grafana-community/grafana \
  --version 14.4.0 --namespace monitoring \
  -f monitoring/grafana-values.yaml \
  --set-file dashboards.kubernetes-lab.kubernetes-overview.json=monitoring/dashboards/kubernetes-overview.json \
  --wait --timeout 10m
```

**Observed result:** Grafana reached `deployed`, revision **3**. API verification returned dashboard title `Kubernetes Lab - Overview`, `uid=k8s-lab-overview`, `panels=5`, `folder=Kubernetes Lab`, and `provisioned=true`:

```bash
read -r -s -p 'Grafana admin password: ' GRAFANA_PASSWORD
echo
curl -fsS -u "admin:${GRAFANA_PASSWORD}" \
  http://127.0.0.1:13000/api/dashboards/uid/k8s-lab-overview \
  | jq '{title: .dashboard.title, uid: .dashboard.uid, panels: (.dashboard.panels|length), folder: .meta.folderTitle, provisioned: .meta.provisioned}'
unset GRAFANA_PASSWORD
```

The dashboard can be opened at `http://127.0.0.1:13000/d/k8s-lab-overview` while port-forwarding is active. **Operational requirement:** repeat `--set-file` on subsequent Helm upgrades, or the dashboard configuration may be omitted from the new release values.

## 12. Persistence and Recovery Test

Restart Grafana's Deployment without reinstalling the release:

```bash
kubectl rollout restart deployment/grafana -n monitoring
kubectl rollout status deployment/grafana -n monitoring --timeout=5m
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
kubectl get pvc grafana -n monitoring
helm history grafana -n monitoring
```

**Observed results:** The rollout succeeded; the new Pod `grafana-788dd9d87-gw7pn` was `1/1 Running` with zero restarts. The PVC remained `Bound` to volume `pvc-7213cbed-4368-4612-832b-ac04ff428433`, capacity 1Gi. Helm stayed at revision **3**, demonstrating that a Kubernetes rollout restart does not create a new Helm release revision.

After the restart, the dashboard API again returned `title=Kubernetes Lab - Overview`, `panels=5`, and `provisioned=true`. This validated the continued availability of the provisioned dashboard across Pod replacement.

## 13. Final Verified State and Operating Commands

| Component | Final observed status |
| --- | --- |
| Prometheus | `deployed`, revision 2, application v3.16.0 |
| Prometheus server | `2/2 Running` |
| kube-state-metrics | `1/1 Running` |
| node-exporter | `1/1 Running` |
| Prometheus targets | 9/9 healthy when tested |
| Prometheus PVC | `Bound`, 2Gi, `standard` |
| Grafana | `deployed`, revision 3, application v13.3.0 |
| Grafana Pod | `1/1 Running` after rollout restart |
| Grafana PVC | `Bound`, 1Gi, `standard` |
| Grafana datasource | `Prometheus`, UID `prometheus`, default and read-only |
| Dashboard | 5 panels, provisioned, persisted across restart |

Useful administration commands:

```bash
helm list -n monitoring
helm history prometheus -n monitoring
helm history grafana -n monitoring
kubectl get deployments,daemonsets,pods,services,pvc -n monitoring
kubectl top pods -n monitoring --containers
kubectl get events -n monitoring --sort-by=.lastTimestamp
kubectl describe node k8s-lab-control-plane | sed -n '/Allocated resources:/,/Events:/p'
```

Stop port-forwarding with `Ctrl+C` in its terminal. Both services remain internal `ClusterIP` Services; this lab did not expose Grafana or Prometheus publicly.

## 14. Lessons Learned and Next Steps

1. Render and lint version-pinned charts before installation. Use server-side dry-run for API validation while accounting for Helm test-hook dependencies.
2. Image manifest architecture matters on Apple Silicon. `ImagePullBackOff` with `no match for platform` is not a resource-limit problem.
3. Investigate Helm timeouts with Pod events and container state; a slow successful download can outlast `--wait --timeout`.
4. A `failed` Helm release can often be repaired with `helm upgrade`, preserving its PVC and revision history.
5. Separate Kubernetes Secret values from version-controlled Helm configuration. The existence of a Secret alone does not prove successful application authentication.
6. Use Grafana provisioning for reproducible datasources and dashboards. File-based provisioning and persistent storage solve different problems.
7. Distinguish Helm release revisions from Kubernetes Deployment revisions and rolling restarts.
8. Monitor both scheduler requests and observed usage; this lab reserved about 90% of allocatable node CPU after installing Grafana.
9. This setup intentionally omitted Alertmanager and Pushgateway. Alerting configuration, monitoring automation, backups, and security hardening remain potential future labs.

**Security note:** This configuration is educational. Production systems require review of image support, workload isolation, authorization, secret management, backup/restore procedures, TLS, access control, and resource availability.
