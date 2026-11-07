# 12 — Kubernetes Metrics and Monitoring

**Laboratory:** K8S-022
**Environment:** kind-k8s-lab
**Kubernetes:** v1.37.0
**Metrics Server:** v0.9.0
**Status:** Practical experiments completed

## 1. Objective

The purpose of this laboratory was to configure Kubernetes resource metrics collection and analyze real CPU and memory usage.

The laboratory covered:

- Inspecting Metrics API availability.
- Installing Metrics Server.
- Troubleshooting Kubernetes kubelet TLS certificate validation.
- Measuring node and Pod CPU and memory usage.
- Comparing actual usage with resource requests and limits.
- Generating controlled CPU load.
- Investigating Linux cgroup v2 CPU throttling.
- Verifying resource quota accounting.
- Cleaning up temporary workloads.

## 2. Laboratory Environment

| Component | Configuration |
|---|---|
| Kubernetes context | kind-k8s-lab |
| Kubernetes node | k8s-lab-control-plane |
| Kubernetes version | v1.37.0 |
| Node OS | Debian GNU/Linux 13 |
| Architecture | ARM64 |
| Container runtime | containerd 2.3.4 |
| Container environment | Colima / Docker |
| Metrics Server namespace | kube-system |
| CPU test namespace | resource-lab |
| Application namespace | lab-apps |

The cluster consisted of a single Kubernetes control-plane node with two available virtual CPUs.

The existing Nginx Deployment had five ready replicas.

## 3. Initial Metrics API Inspection

Commands:

```bash
kubectl config current-context
kubectl get nodes -o wide

kubectl get deployments -A | grep -i metrics || true
kubectl get pods -A | grep -i metrics || true

kubectl get apiservice v1beta1.metrics.k8s.io

kubectl top nodes
kubectl top pods -A
```

Results:

```text
apiservices.apiregistration.k8s.io
"v1beta1.metrics.k8s.io" not found
```

```text
error: Metrics API not available
```

Neither the Metrics Server Deployment nor its Pods were present.

**Conclusion:** Metrics API was not available because Metrics Server had not been installed.

## 4. Metrics Server Installation

The selected version was Metrics Server v0.9.0.

Official project:

https://github.com/kubernetes-sigs/metrics-server

Download:

```bash
mkdir -p manifests/metrics-server

curl -fL \
  https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.9.0/components.yaml \
  -o manifests/metrics-server/components.yaml
```

Manifest validation:

```bash
kubectl apply --dry-run=server \
  -f manifests/metrics-server/components.yaml
```

Installation:

```bash
kubectl apply \
  -f manifests/metrics-server/components.yaml
```

The manifest installed the Metrics Server Deployment, Service, APIService and associated Kubernetes resources.

## 5. Initial Deployment Status

Commands:

```bash
kubectl get deployment metrics-server -n kube-system

kubectl get pods -n kube-system \
  -l k8s-app=metrics-server -o wide

kubectl get apiservice v1beta1.metrics.k8s.io
```

Initial status:

```text
metrics-server   0/1
```

```text
ContainerCreating
```

```text
False (MissingEndpoints)
```

At first, the container image was still being downloaded.

The Metrics Server Pod subsequently reached `Running`, but did not become Ready.

## 6. Readiness Probe Failure

Pod inspection:

```bash
kubectl describe pods \
  -n kube-system \
  -l k8s-app=metrics-server
```

Observed event:

```text
Warning Unhealthy
Readiness probe failed:
HTTP probe failed with statuscode: 500
```

The Deployment remained unavailable.

The failure required further investigation of Metrics Server logs.

## 7. TLS Certificate Troubleshooting

Logs:

```bash
kubectl logs \
  -n kube-system \
  deployment/metrics-server \
  --tail=100 \
  --timestamps
```

Observed error:

```text
Failed to scrape node

Get "https://172.18.0.2:10250/metrics/resource":

tls: failed to verify certificate:

x509: cannot validate certificate for 172.18.0.2
because it doesn't contain any IP SANs
```

Metrics Server attempted to scrape kubelet resource metrics over HTTPS using the node's internal IP address.

The kubelet certificate did not contain a matching IP Subject Alternative Name (SAN).

Consequently, the TLS certificate identity validation failed.

An additional readiness error appeared:

```text
Failed probe
probe="metric-storage-ready"
err="no metrics to serve"
```

Because resource metrics could not be collected, Metrics Server had no measurements to serve.

This was a certificate validation problem, not a container image or CPU architecture problem.

## 8. Metrics Server TLS Configuration

The laboratory used the following Metrics Server arguments:

```yaml
args:
  - --cert-dir=/tmp
  - --secure-port=10250
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
  - --kubelet-use-node-status-port
  - --metric-resolution=15s
  - --kubelet-insecure-tls
```

The additional argument was:

```text
--kubelet-insecure-tls
```

This disables kubelet certificate verification for Metrics Server connections.

**Security warning:** This setting was applied only to an isolated local kind laboratory.

It must not be treated as a recommended production configuration.

A production configuration should use appropriate trusted kubelet serving certificates containing valid Subject Alternative Names.

The modified manifest was saved to:

`manifests/metrics-server/components.yaml`

The updated configuration was deployed:

```bash
kubectl apply \
  -f manifests/metrics-server/components.yaml

kubectl rollout status \
  deployment/metrics-server \
  -n kube-system \
  --timeout=120s
```

## 9. Successful Metrics Server Startup

After applying the laboratory TLS workaround:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=metrics-server

kubectl get apiservice v1beta1.metrics.k8s.io
```

Results:

```text
metrics-server   1/1 Running
```

```text
v1beta1.metrics.k8s.io   True
```

The container had zero restarts.

The new logs no longer contained the previous kubelet certificate validation errors.

The Metrics API became available.

## 10. First Node Metrics

Command:

```bash
kubectl top nodes
```

Initial measurements:

| Node | CPU | CPU % | Memory | Memory % |
|---|---|---|---|---|
| k8s-lab-control-plane | 184m | 9% | 761Mi | 19% |

This measurement confirmed that Metrics Server was collecting resource data successfully.

Metrics Server measurements represent observed resource usage, not resource reservations.

## 11. Nginx Pod Metrics

Command:

```bash
kubectl top pods -n lab-apps
```

Observed results:

| Pod | CPU | Memory |
|---|---|---|
| nginx-deployment-...-b2z5l | 1m | 3Mi |
| nginx-deployment-...-b4ck6 | 1m | 3Mi |
| nginx-deployment-...-jsv2z | 1m | 3Mi |
| nginx-deployment-...-pnx9l | 1m | 3Mi |
| nginx-deployment-...-wzrbg | 1m | 3Mi |
| **Total** | **5m** | **15Mi** |

All five Nginx replicas showed similarly low resource consumption.

The application was running without a significant test workload.

## 12. Comparing Resource Requests, Limits and Usage

Nginx configuration:

```bash
kubectl get deployment nginx-deployment \
  -n lab-apps \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\n"}{.resources}{"\n"}{end}'
```

Observed container resources:

```json
{
  "limits": {
    "cpu": "250m",
    "memory": "128Mi"
  },
  "requests": {
    "cpu": "50m",
    "memory": "64Mi"
  }
}
```

Measured values per Nginx Pod:

| Resource | Request | Limit | Usage |
|---|---|---|---|
| CPU | 50m | 250m | 1m |
| Memory | 64Mi | 128Mi | 3Mi |

CPU usage represented approximately 2% of the request.

Memory usage represented approximately 4.69% of the request.

The results demonstrated the difference between declared scheduling requirements and actual consumption.

However, low resource usage without significant traffic does not establish appropriate resource requests for production conditions.

## 13. Metrics Server Resource Consumption

Metrics Server configuration:

```bash
kubectl get deployment metrics-server \
  -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

Result:

```json
{
  "requests": {
    "cpu": "100m",
    "memory": "200Mi"
  }
}
```

Measured usage:

```bash
kubectl top pods -n kube-system \
  -l k8s-app=metrics-server
```

| Resource | Request | Observed usage |
|---|---|---|
| CPU | 100m | 5m |
| Memory | 200Mi | 17Mi |

The installation had no explicit CPU or memory limits.

In the small laboratory cluster, Metrics Server consumed comparatively few resources during the measurement.

## 14. Node Resource Allocation

Command:

```bash
kubectl describe node k8s-lab-control-plane |
  grep -A 12 '^Allocated resources:'
```

Observed allocation:

| Resource | Requests | Limits |
|---|---|---|
| CPU | 1300m (65%) | 1250m (62%) |
| Memory | 810Mi (20%) | 980Mi (25%) |

Actual usage during the comparison:

| Resource | Usage |
|---|---|
| CPU | 205m (10%) |
| Memory | 771Mi (19%) |

The node had considerably higher CPU requests than actual CPU usage.

Kubernetes scheduling considers declared requests rather than instantaneous CPU usage.

## 15. Controlled CPU Load Experiment

The laboratory used an isolated Pod:

`manifests/cpu-load-test.yaml`

Test configuration:

| Parameter | Value |
|---|---|
| Namespace | resource-lab |
| Pod | cpu-load-test |
| Image | busybox:1.36 |
| CPU request | 50m |
| CPU limit | 200m |
| Memory request | 32Mi |
| Memory limit | 128Mi |
| Load duration | 180 seconds |

The command executed inside the container:

```sh
echo "K8S-022 CPU load test started"

yes > /dev/null &
WORKER_PID=$!

sleep 180

kill "$WORKER_PID"
wait "$WORKER_PID" 2>/dev/null || true

echo "CPU load test completed"
sleep 3600
```

The `yes` process generated continuous CPU activity.

The final `sleep` preserved the container for post-test inspection.

## 16. Starting the Test Pod

Validation:

```bash
kubectl apply --dry-run=server \
  -f manifests/cpu-load-test.yaml
```

Creation:

```bash
kubectl apply -f manifests/cpu-load-test.yaml
```

Readiness verification:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/cpu-load-test \
  -n resource-lab \
  --timeout=120s
```

Observed:

```text
pod/cpu-load-test condition met
```

ResourceQuota usage:

| Resource | Used | Hard |
|---|---|---|
| requests.cpu | 50m | 300m |
| limits.cpu | 200m | 1000m |
| requests.memory | 32Mi | 256Mi |
| limits.memory | 128Mi | 768Mi |
| pods | 1 | 4 |

## 17. First Metrics Collection Delay

Immediately after creating the Pod:

```bash
kubectl top pod cpu-load-test -n resource-lab
```

Result:

```text
Error from server (NotFound):
podmetrics.metrics.k8s.io
"resource-lab/cpu-load-test" not found
```

The Pod had existed for approximately 12 seconds.

Metrics Server was configured with a 15-second metric resolution.

The Pod was Running and Ready, but its first metric was not yet available.

This illustrated that newly created Pods may not appear immediately in Metrics API results.

## 18. CPU Usage Under Load

A later measurement:

```bash
kubectl top pod cpu-load-test -n resource-lab
```

Result:

```text
NAME            CPU(cores)   MEMORY(bytes)
cpu-load-test   201m         0Mi
```

Container-level measurement:

```bash
kubectl top pod cpu-load-test \
  -n resource-lab \
  --containers
```

Result:

```text
POD             NAME       CPU(cores)  MEMORY(bytes)
cpu-load-test   cpu-load   201m        0Mi
```

CPU usage was approximately equal to the configured 200m limit.

The 1m difference is consistent with short-window measurement and reporting effects.

The `0Mi` memory value was the displayed metric value, not proof of zero memory consumption.

## 19. Confirming the CPU Load Process

Command:

```bash
kubectl exec -n resource-lab \
  cpu-load-test -- ps
```

Observed process:

```text
PID  USER  TIME  COMMAND
14   root  0:22  yes
```

The generator was active during the measurement.

Logs:

```text
K8S-022 CPU load test started
```

Node usage during the experiment:

```text
CPU:    327m (16%)
Memory: 775Mi (19%)
```

The node CPU measurement was higher than the earlier idle measurement.

Node-level metrics also include other workloads and system processes.

## 20. Linux cgroup v2 CPU Configuration

The container exposed:

```text
/sys/fs/cgroup/cpu.max
/sys/fs/cgroup/cpu.stat
```

Commands:

```bash
kubectl exec -n resource-lab cpu-load-test -- \
  cat /sys/fs/cgroup/cpu.max

kubectl exec -n resource-lab cpu-load-test -- \
  cat /sys/fs/cgroup/cpu.stat
```

The CPU limit configuration was:

```text
20000 100000
```

Meaning:

- CPU quota: 20,000 microseconds.
- CPU period: 100,000 microseconds.
- CPU allocation: 20% of one CPU.

This matched the Kubernetes CPU limit of 200m.

## 21. CPU Throttling Statistics

Observed cgroup v2 counters:

```text
usage_usec 36039483
user_usec 35695110
system_usec 344373
core_sched.force_idle_usec 0
nr_periods 1807
nr_throttled 282
throttled_usec 6266182
nr_bursts 0
burst_usec 0
```

Key metrics:

| Counter | Value |
|---|---:|
| nr_periods | 1807 |
| nr_throttled | 282 |
| throttled_usec | 6266182 |
| usage_usec | 36039483 |

The proportion of periods with throttling was:

```text
282 / 1807 × 100 ≈ 15.6%
```

Approximately 15.6% of recorded CPU quota periods experienced throttling.

The accumulated CPU usage counter represented approximately 36.04 seconds of CPU time.

The accumulated throttling counter represented approximately 6.27 seconds of reported throttling time.

**Important:** The throttling counter must not be interpreted as an exact amount of lost application CPU execution time.

The measurements demonstrated that the Linux kernel enforced the configured CPU limit.

## 22. Comparing CPU Usage Before and After the Test

During load:

```text
cpu-load-test   201m   0Mi
```

After load:

```text
cpu-load-test   0m     0Mi
```

The `yes` process was no longer present in the process list.

The logs contained:

```text
K8S-022 CPU load test started
CPU load test completed
```

The test confirmed that CPU consumption dropped after the load generator finished.

The Pod remained Running because the shell script executed a final sleep command.

## 23. Test Cleanup

Command:

```bash
kubectl delete pod cpu-load-test \
  -n resource-lab \
  --wait=true \
  --timeout=120s
```

Result:

```text
pod "cpu-load-test" deleted from resource-lab namespace
```

Namespace inspection:

```bash
kubectl get pods -n resource-lab
```

Result:

```text
No resources found in resource-lab namespace.
```

ResourceQuota usage returned to zero:

| Resource | Used | Hard |
|---|---|---|
| requests.cpu | 0 | 300m |
| limits.cpu | 0 | 1000m |
| requests.memory | 0 | 256Mi |
| limits.memory | 0 | 768Mi |
| pods | 0 | 4 |

The namespace policies were retained.

## 24. Final Metrics Server Verification

Commands:

```bash
kubectl get deployment metrics-server -n kube-system

kubectl get apiservice v1beta1.metrics.k8s.io

kubectl top nodes

kubectl get deployment nginx-deployment -n lab-apps
```

Final results:

| Component | Result |
|---|---|
| Metrics Server | 1/1 Ready |
| Metrics API | Available |
| Node CPU usage | 190m (9%) |
| Node memory usage | 763Mi (19%) |
| Nginx Deployment | 5/5 Ready |

The test Pod was removed without affecting existing workloads.

## 25. Security Considerations

The Metrics Server manifest contains:

```text
--kubelet-insecure-tls
```

This option disables kubelet serving-certificate verification.

It was required as a laboratory workaround because the kubelet certificate did not contain the appropriate IP SAN.

The option is not recommended for production deployments.

In production, configure trusted kubelet serving certificates with valid identity information and maintain TLS verification.

The Metrics Server installation also uses Kubernetes RBAC and an aggregated APIService.

Changes to its permissions and network exposure should be reviewed before production use.

## 26. Monitoring Architecture

The resource metrics path in this laboratory was:

```text
Kubernetes Pods / Containers
          |
          v
        kubelet
          |
          v
     Metrics Server
          |
          v
   metrics.k8s.io API
          |
          v
       kubectl top
```

Metrics Server provides recent CPU and memory resource metrics.

It is not a replacement for a complete observability platform providing long-term metric retention, dashboards and alerting.

Possible future laboratory topics include:

- Prometheus.
- Grafana.
- Kubernetes monitoring dashboards.
- CPU and memory alerting.
- Horizontal Pod Autoscaler.
- Comparing application performance under different CPU limits.

## 27. Operational Lessons Learned

1. A working Kubernetes cluster does not necessarily include Metrics Server.
2. `kubectl top` depends on an available resource metrics API.
3. Metrics Server must communicate with kubelets to collect measurements.
4. Kubelet serving-certificate identity mismatches can prevent metric collection.
5. A Running container may remain NotReady when essential functionality fails.
6. Logs and readiness probes are essential troubleshooting tools.
7. Laboratory TLS workarounds must not become production defaults.
8. Metrics Server can run successfully in a kind cluster on ARM64.
9. Kubernetes requests are scheduling inputs, not maximum usage limits.
10. CPU limits are enforced using Linux resource-control mechanisms.
11. Actual CPU usage can exceed the CPU request.
12. Kubernetes reports CPU usage in millicores.
13. ResourceQuota accounts for declared resources rather than runtime metrics.
14. Metrics Server may need time to produce initial metrics for new Pods.
15. cgroup v2 provides low-level resource accounting and throttling counters.
16. CPU throttling can be detected by examining `nr_throttled`.
17. A throttling-period percentage is not the same as a percentage of lost performance.
18. Low idle CPU usage does not establish appropriate production resource requests.
19. Resource experiments should be isolated from existing applications.
20. Temporary workloads should be removed after experiments.

## 28. Laboratory Result

**Practical experiments completed successfully.**

Metrics Server v0.9.0 was installed and configured.

A kubelet TLS certificate validation problem was identified from application logs.

A laboratory-specific configuration change restored Metrics Server functionality.

The resource metrics API became available.

Node and Pod CPU and memory measurements were collected.

Actual resource usage was compared with Kubernetes resource requests and limits.

A controlled CPU load test demonstrated consumption near a configured 200m CPU limit.

Linux cgroup v2 counters confirmed CPU throttling.

CPU usage was observed both during the load test and after its completion.

The temporary test Pod was deleted.

ResourceQuota usage returned to zero.

Metrics Server remained healthy.

The existing Nginx Deployment retained all five ready replicas.

The next repository tasks are README updates, final validation, and Git publication.
