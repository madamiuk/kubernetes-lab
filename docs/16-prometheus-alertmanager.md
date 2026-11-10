
# 16 — Prometheus Alerting and Alertmanager: Kubernetes Health Monitoring

**Laboratory:** K8S-026

**Environment:** kind-k8s-lab

**Kubernetes:** v1.37.0

**Helm:** v4.3.1

**Prometheus:** v3.16.0

**Prometheus Helm chart:** 29.45.1

**Alertmanager:** v0.34.2

**Alertmanager Helm subchart:** 2.3.1

**Status:** Practical experiments completed

## 1. Objective

This laboratory extended the existing Kubernetes monitoring stack with Prometheus alerting rules and Alertmanager.

The objectives were to:

- Deploy Alertmanager using the existing Prometheus Helm release.
- Configure resource requests and limits for Alertmanager and its configuration reloader.
- Verify Prometheus-to-Alertmanager communication.
- Create and validate a controlled test alert.
- Test alert routing and Silence management through the Alertmanager API.
- Implement Kubernetes health alerting rules using real kube-state-metrics data.
- Simulate a Pod readiness failure and verify the complete alert lifecycle.
- Remove temporary test alerts while preserving reusable monitoring rules.

## 2. Architecture

The alerting pipeline uses the existing monitoring infrastructure:

```text
Kubernetes API
      |
      v
kube-state-metrics
      |
      v
Prometheus
      |
      +-- Alert evaluation
      |      |
      |      +-- inactive
      |      +-- pending
      |      +-- firing
      |
      v
Alertmanager
      |
      +-- Routing
      +-- Grouping
      +-- Silences
      |
      v
default-receiver
```

All monitoring components run in the `monitoring` namespace.

The laboratory uses a single-node kind cluster with limited CPU and memory resources.

No external notification integration was configured.

## 3. Alertmanager Deployment

Alertmanager was enabled through the existing Helm values file:

`monitoring/prometheus-values.yaml`

The configuration uses:

- One Alertmanager replica.
- Ephemeral storage (`persistence.enabled: false`).
- CPU request of 25m and memory request of 32Mi.
- CPU limit of 100m and memory limit of 128Mi.
- An Alertmanager configuration reloader with explicit resource limits.
- ARM64-compatible configuration reloader image `v0.94.2`.

The release was upgraded using:

```bash
helm upgrade prometheus \
  /tmp/k8s-025-charts/prometheus \
  --namespace monitoring \
  --values monitoring/prometheus-values.yaml \
  --wait \
  --timeout 10m
```

Deployment verification:

```bash
kubectl get pods -n monitoring
kubectl get services -n monitoring
helm list -n monitoring
```

The Alertmanager Pod reached:

```text
prometheus-alertmanager-0   2/2   Running
```

The Alertmanager service exposes port `9093`.

### Persistence limitation

Alertmanager persistence is disabled to reduce resource consumption in this laboratory.

Silences and notification state are not guaranteed to survive Pod replacement.

Persistent Alertmanager storage should be considered for production environments.

## 4. Prometheus and Alertmanager Integration

Prometheus discovers Alertmanager through Kubernetes service discovery.

The integration was verified using the Prometheus API:

```bash
kubectl exec -n monitoring \
  deployment/prometheus-server \
  -c prometheus-server -- \
  wget -qO- \
  'http://127.0.0.1:9090/api/v1/alertmanagers'
```

The API reported an active Alertmanager target using port `9093`.

The Alertmanager API was also verified directly:

```bash
kubectl exec -n monitoring \
  prometheus-alertmanager-0 \
  -c alertmanager -- \
  wget -qO- \
  'http://127.0.0.1:9093/api/v2/alerts'
```

## 5. Controlled Test Alert

The first test used a temporary Prometheus alert:

```yaml
- alert: KubernetesLabTestAlert
  expr: vector(1)
  for: 2m
  labels:
    severity: warning
    environment: lab
  annotations:
    summary: "Kubernetes laboratory test alert"
    description: "Controlled test alert for validating Prometheus and Alertmanager integration."
```

The expression `vector(1)` always evaluates to a nonzero value.

It was used only for integration testing.

After the configured `for: 2m` interval, Prometheus reported:

```text
KubernetesLabTestAlert   firing
```

Alertmanager received the alert and reported:

```text
state: active
receiver: default-receiver
```

The test confirmed that Prometheus successfully forwarded firing alerts to Alertmanager.

## 6. Alertmanager Silence Management

A Silence suppresses notifications for alerts matching selected labels.

It does not change the underlying Prometheus rule evaluation.

A test Silence was created using the Alertmanager HTTP API.

### Access Alertmanager locally

```bash
kubectl port-forward \
  -n monitoring \
  svc/prometheus-alertmanager \
  9093:9093
```

### Create a Silence

The second Silence experiment used a 30-minute validity interval.

```bash
SILENCE_START=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
SILENCE_END=$(date -u -v+30M +"%Y-%m-%dT%H:%M:%SZ")

jq -n \
  --arg start "$SILENCE_START" \
  --arg end "$SILENCE_END" \
  '{
    matchers: [
      {
        name: "alertname",
        value: "KubernetesLabTestAlert",
        isRegex: false,
        isEqual: true
      }
    ],
    startsAt: $start,
    endsAt: $end,
    createdBy: "kubernetes-lab",
    comment: "K8S-026 silence suppression verification"
  }' > /tmp/k8s-026-silence.json
```

Submit the Silence:

```bash
curl -fsS -X POST \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/k8s-026-silence.json \
  http://127.0.0.1:9093/api/v2/silences \
  | jq .
```

### Verify suppression

The API confirmed:

```text
Silence status: active
Alertmanager alert state: suppressed
silencedBy: [Silence ID]
```

A previous shorter Silence had already expired before its suppression could be verified. Repeating the experiment with a longer interval allowed direct confirmation.

### Expire a Silence manually

```bash
curl -fsS -X DELETE \
  "http://127.0.0.1:9093/api/v2/silence/$SILENCE_ID"
```

The Silence status changed to:

```text
expired
```

The alert returned to:

```text
state: active
silencedBy: []
```

This verified the complete Silence lifecycle.

## 7. Kubernetes Health Alert Rules

Two reusable alerts were implemented in the `kubernetes-health` rule group.

### KubernetesPodNotReady

```promql
(
  kube_pod_status_ready{condition="false"} == 1
)
and on (namespace, pod, uid)
(
  kube_pod_status_phase{phase=~"Pending|Running"} == 1
)
```

Configuration:

```text
alert: KubernetesPodNotReady
for: 5m
severity: warning
environment: lab
```

This rule detects Pods that remain not ready while in the `Pending` or `Running` phase.

### KubernetesNodeMemoryPressure

```promql
kube_node_status_condition{
  condition="MemoryPressure",
  status="true"
} == 1
```

Configuration:

```text
alert: KubernetesNodeMemoryPressure
for: 5m
severity: warning
environment: lab
```

This rule detects a node reporting the Kubernetes `MemoryPressure` condition.

Both expressions were tested against live Prometheus metrics before deployment.

Under normal conditions, both queries returned no matching series.

## 8. Controlled Pod Readiness Failure

A temporary Pod manifest was created:

`monitoring/tests/pod-not-ready.yaml`

The Pod uses Nginx with a deliberately failing readiness probe:

```yaml
readinessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - exit 1
  initialDelaySeconds: 5
  periodSeconds: 10
```

The manifest defines small CPU and memory requests and limits.

A liveness probe was intentionally omitted.

This allowed the container to continue running while Kubernetes reported the Pod as not ready.

### Deploy the test Pod

```bash
kubectl apply \
  -f monitoring/tests/pod-not-ready.yaml
```

### Verify its state

```bash
kubectl get pod pod-not-ready-test \
  -n monitoring

kubectl describe pod pod-not-ready-test \
  -n monitoring
```

Observed status:

```text
pod-not-ready-test   0/1   Running   0
```

The Kubernetes events included repeated:

```text
Readiness probe failed
```

The test container remained running without restarts.

## 9. Alert Lifecycle Verification

Prometheus reported:

```text
kube_pod_status_ready{
  namespace="monitoring",
  pod="pod-not-ready-test",
  condition="false"
} = 1
```

The `KubernetesPodNotReady` alert entered the `pending` state.

Its observed activation time was:

```text
2026-11-10T07:31:48Z
```

After the configured five-minute interval, the alert entered the `firing` state.

Alertmanager received the alert with:

```text
alertname: KubernetesPodNotReady
namespace: monitoring
pod: pod-not-ready-test
severity: warning
state: active
receiver: default-receiver
startsAt: 2026-11-10T07:36:48Z
```

The timestamps confirmed the configured five-minute alert delay.

### Remove the test Pod

```bash
kubectl delete \
  -f monitoring/tests/pod-not-ready.yaml
```

The Pod was successfully removed.

Prometheus initially continued to show `firing`, then reported:

```text
KubernetesPodNotReady   inactive
activeAlerts: []
```

The Alertmanager API subsequently returned no current `KubernetesPodNotReady` alert.

The complete lifecycle was verified:

```text
inactive -> pending -> firing -> inactive
```

## 10. Final Configuration and Cleanup

The temporary `KubernetesLabTestAlert` rule was removed from the Helm values file.

The two Kubernetes health rules were retained.

The configuration was validated with:

```bash
helm lint /tmp/k8s-025-charts/prometheus \
  -f monitoring/prometheus-values.yaml
```

Manifest rendering:

```bash
helm template prometheus \
  /tmp/k8s-025-charts/prometheus \
  --namespace monitoring \
  -f monitoring/prometheus-values.yaml \
  > /tmp/k8s-026-prometheus-final.yaml
```

Final deployment:

```bash
helm upgrade prometheus \
  /tmp/k8s-025-charts/prometheus \
  --namespace monitoring \
  --values monitoring/prometheus-values.yaml \
  --wait \
  --timeout 10m
```

Final Helm release:

```text
Release: prometheus
Revision: 6
Status: deployed
Chart: prometheus-29.45.1
Application: v3.16.0
```

The final Prometheus rule state was:

```text
kubernetes-health
  KubernetesPodNotReady         inactive
  KubernetesNodeMemoryPressure  inactive
```

The temporary test rule was absent from:

- The Kubernetes ConfigMap.
- The mounted Prometheus rules file.
- The Prometheus rules API.

No Prometheus restart was required.

The test Pod was removed, but its manifest remains in the repository for future reproducible experiments.

## 11. Troubleshooting and Operational Observations

### Configuration reload visibility

Immediately after some Helm upgrades, the Prometheus rules API still displayed the previous rule configuration.

The ConfigMap and mounted rule file were checked before considering a restart.

Subsequent API queries confirmed that the updated rules had been loaded successfully.

### Alert evaluation intervals

Prometheus evaluates rules periodically.

Changes in Pod state or alert configuration are therefore not always reflected immediately in the rules API.

### Pending versus firing

The `for` interval prevents a newly detected condition from immediately generating a firing alert.

During the Pod readiness experiment, the alert remained pending before entering the firing state.

### Alertmanager versus Prometheus states

Prometheus uses states such as `inactive`, `pending`, and `firing`.

Alertmanager reports alert handling states such as `active` and `suppressed`.

Silence management affects Alertmanager notification handling, not Prometheus rule evaluation.

### Resource constraints

The laboratory runs on a single Kubernetes node with limited CPU and memory.

Small resource requests were used for the test Pod and Alertmanager.

No additional load-generating workload was required for the readiness experiment.

## 12. Result

The laboratory successfully demonstrated:

- Alertmanager deployment and Prometheus integration.
- Temporary alert rules for controlled integration testing.
- Silence creation, suppression, and manual expiration.
- Kubernetes health alert rules based on real metrics.
- A controlled readiness failure without container restarts.
- The transition from `pending` to `firing`.
- Alertmanager reception and routing of a real Kubernetes alert.
- Automatic alert resolution following test Pod removal.
- Cleanup of temporary rules and verification of the final Helm configuration.

The cluster now retains two reusable health-monitoring alerts, both inactive under the observed healthy conditions.

**Laboratory K8S-026 completed successfully.**
