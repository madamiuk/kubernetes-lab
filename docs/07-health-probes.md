# 07 — Kubernetes Health Probes

## 1. Objective

Configure and test Kubernetes readiness, liveness, and startup probes using Nginx.

Demonstrate the operational differences between removing an unhealthy Pod from Service endpoints, restarting a failed container, and allowing applications with slow startup times to initialize successfully.

**Related laboratory:** K8S-017.

## 2. Laboratory Environment

| Component | Value |
|---|---|
| Kubernetes cluster | k8s-lab |
| Namespace | lab-apps |
| Application Deployment | nginx-deployment |
| Application replicas | 5 |
| Nginx image | nginx:stable-alpine |
| Observed Nginx version | 1.30.6 |
| Service | nginx-service |
| ClusterIP | 10.96.201.49 |
| Primary manifest | manifests/nginx-deployment.yaml |
| Readiness test manifest | manifests/readiness-test.yaml |
| Startup test manifest | manifests/startup-test.yaml |

The tests were performed on a local single-node Kubernetes cluster running in kind.

## 3. Initial State

Before the laboratory, the Nginx Deployment had no health probes configured.

Verification command:

```bash
kubectl get deployment nginx-deployment \
  -n lab-apps \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\nreadiness: "}{.readinessProbe}{"\nliveness: "}{.livenessProbe}{"\nstartup: "}{.startupProbe}{"\n"}{end}'
```

Observed state:

```text
nginx
readiness:
liveness:
startup:
```

All five Pods were Running and Ready, with zero restarts.

Without an explicit readiness probe, a running container can be considered ready even if the application is not correctly serving HTTP traffic.

## 4. Health Probe Concepts

### Readiness Probe

Determines whether a container is ready to receive traffic.

When readiness fails, Kubernetes marks the Pod as not ready and updates the readiness state of its Service endpoint.

Readiness failures do not directly restart containers.

### Liveness Probe

Determines whether a container should be restarted.

After the configured number of consecutive failures, kubelet restarts the container according to its restart policy.

### Startup Probe

Determines whether an application has completed its startup sequence.

Until the startup probe succeeds, liveness and readiness probing is suppressed.

If the startup probe exceeds its failure threshold, kubelet restarts the container.

## 5. Main Deployment Configuration

File:

`manifests/nginx-deployment.yaml`

The following probes were added to the Nginx container:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 12

readinessProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
  successThreshold: 1

livenessProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

The image, five replicas, resource requests, and resource limits were preserved.

## 6. YAML Troubleshooting

The first attempts to modify the Deployment manifest produced YAML parsing errors.

Examples:

```text
yaml: line 23: did not find expected '-' indicator
yaml: line 24: mapping values are not allowed in this context
```

The problems were caused by incorrect indentation of `containerPort`, `startupProbe`, and the HTTP probe fields.

The configuration was corrected by placing `ports`, `startupProbe`, `readinessProbe`, `livenessProbe`, and `resources` at the same container-field indentation level.

Vim paste mode was used to prevent automatic indentation from affecting pasted YAML:

```vim
:set paste
```

After pasting:

```vim
:set nopaste
:wq
```

Server-side validation:

```bash
kubectl apply --dry-run=server \
  -f manifests/nginx-deployment.yaml
```

Successful result:

```text
deployment.apps/nginx-deployment configured (server dry run)
```

## 7. Deployment of Health Probes

The validated manifest was applied:

```bash
kubectl apply -f manifests/nginx-deployment.yaml
```

The rollout was checked:

```bash
kubectl rollout status \
  deployment/nginx-deployment \
  -n lab-apps \
  --timeout=180s
```

Changing the Pod template created a new ReplicaSet:

```text
nginx-deployment-5c7448d7d6
```

Five Pods were Running and Ready with zero restarts immediately after rollout.

The older ReplicaSets remained present with zero desired replicas.

## 8. Verification of Configured Probes

Command:

```bash
kubectl describe deployment nginx-deployment \
  -n lab-apps |
  grep -E 'Liveness:|Readiness:|Startup:'
```

Observed settings:

| Probe | Period | Timeout | Failure threshold |
|---|---:|---:|---:|
| Startup | 5 seconds | 2 seconds | 12 |
| Readiness | 5 seconds | 2 seconds | 2 |
| Liveness | 10 seconds | 2 seconds | 3 |

All probes checked HTTP `/` on port 80.

## 9. Initial Endpoint and Pod Verification

EndpointSlice inspection:

```bash
kubectl get endpointslices \
  -n lab-apps \
  -l kubernetes.io/service-name=nginx-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{"\n"}{end}'
```

Observed addresses:

```text
10.244.0.28 ready=true
10.244.0.27 ready=true
10.244.0.29 ready=true
10.244.0.30 ready=true
10.244.0.31 ready=true
```

All five application Pods had `Ready=True`, and initially none had restarted.

## 10. Liveness Probe Failure Experiment

### Objective
Simulate an HTTP failure without deliberately terminating the Nginx master process.

### Selected Pod

```text
nginx-deployment-5c7448d7d6-6lrx8
```

The Nginx master process ran as root, while worker processes ran as the `nginx` user.

### Failure Injection

```bash
kubectl exec -n lab-apps \
  nginx-deployment-5c7448d7d6-6lrx8 \
  -- chmod 700 /usr/share/nginx/html
```

The change prevented unprivileged worker processes from accessing the HTML directory.

HTTP requests began returning status 403.

### Readiness Failure

Observed Pod conditions:

```text
Ready=False
ContainersReady=False
```

Kubelet reported failed readiness checks with HTTP 403.

### Liveness Failure and Recovery

The liveness probe failed three consecutive times.

Kubelet reported:

```text
Container nginx failed liveness probe, will be restarted
```

The container restart count increased from 0 to 1.

After restarting, the directory permissions returned to `755`, restoring access to the HTML content.

The Pod returned to `Ready=True`, while its name and IP address remained unchanged.

**Result:** Liveness Probe detected the unhealthy application and restarted its container.

## 11. Isolated Readiness Experiment

To isolate readiness behavior from liveness restarts, a separate Pod and Service were deployed.

Manifest:

`manifests/readiness-test.yaml`

Resources:

- Pod: `nginx-readiness-test`
- Service: `nginx-readiness-test`

The test Pod used the label `app=nginx-readiness-test`, keeping it separate from the main Deployment.

The Pod contained a readiness probe but no liveness probe.

### Deployment and Verification

```bash
kubectl apply --dry-run=server -f manifests/readiness-test.yaml
kubectl apply -f manifests/readiness-test.yaml

kubectl wait \
  --for=condition=Ready \
  pod/nginx-readiness-test \
  -n lab-apps \
  --timeout=120s
```

Initial state:

```text
Pod IP:      10.244.0.32
Ready:       True
Restarts:    0
Service IP:  10.96.226.137
Endpoint:    10.244.0.32 ready=true
```

## 12. Isolated Readiness Failure

The diagnostic Pod was deliberately made unable to serve its HTML content:

```bash
kubectl exec -n lab-apps nginx-readiness-test -- \
  chmod 700 /usr/share/nginx/html
```

The readiness probe reported HTTP 403.

Observed Pod conditions:

```text
Ready=False
ContainersReady=False
```

EndpointSlice state:

```text
10.244.0.32 ready=false
```

Restart count:

```text
0
```

Kubelet recorded five readiness failures caused by HTTP 403.

The main Nginx Deployment remained healthy with five available replicas.

**Result:** The readiness probe removed the diagnostic Pod from the Service's ready endpoints without restarting the container.

## 13. Recovery from Readiness Failure

The directory permissions were restored:

```bash
kubectl exec -n lab-apps nginx-readiness-test -- \
  chmod 755 /usr/share/nginx/html
```

HTTP verification:

```bash
kubectl exec -n lab-apps nginx-readiness-test -- \
  wget -S -O /dev/null http://127.0.0.1/
```

Observed response:

```text
HTTP/1.1 200 OK
Server: nginx/1.30.6
```

Kubernetes readiness verification:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/nginx-readiness-test \
  -n lab-apps \
  --timeout=60s
```

The Pod returned to `Ready=True`.

EndpointSlice:

```text
10.244.0.32 ready=true
```

The restart count remained zero.

The complete observed sequence was:

```text
Ready=True
    |
    | HTTP 403
    v
Ready=False
    |
    | HTTP 200 restored
    v
Ready=True
```

After verification, the diagnostic resources were removed:

```bash
kubectl delete -f manifests/readiness-test.yaml
```

## 14. Startup Probe Experiment

### Objective

Simulate an application requiring additional startup time and verify that Startup Probe prevents premature liveness restarts.

Manifest:

`manifests/startup-test.yaml`

Pod:

`nginx-startup-test`

The container startup sequence included an intentional 25-second delay:

```yaml
command:
  - /bin/sh
  - -c
args:
  - |
    echo "Simulating slow application startup..."
    sleep 25
    echo "Starting Nginx..."
    exec nginx -g 'daemon off;'
```

The startup probe checked HTTP `/` on port 80 using:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 12
```

Readiness and liveness probes were also configured.

The liveness probe used a five-second period and a failure threshold of three.

During the startup phase, Kubernetes suppressed readiness and liveness checks until the startup probe succeeded.

## 15. Startup Probe Results

### Validation and Deployment

```bash
kubectl apply --dry-run=server -f manifests/startup-test.yaml
kubectl apply -f manifests/startup-test.yaml
```

The container started at:

```text
2026-11-01T06:28:17Z
```

Nginx began running approximately 25 seconds later:

```text
2026-11-01T06:28:42Z
```

### Startup Probe Events

Kubelet recorded five failed startup checks:

```text
Warning Unhealthy (x5)
Startup probe failed:
connect: connection refused
```

The probe attempted to connect to:

```text
10.244.0.33:80
```

The container was not restarted.

### Final State

```text
READY:    1/1
STATUS:   Running
RESTARTS: 0
```

The Pod conditions included:

```text
Ready=True
ContainersReady=True
```

Nginx logs subsequently showed successful HTTP 200 requests from `kube-probe/1.37`.

**Result:** The startup probe tolerated the deliberate 25-second initialization period, allowing Nginx to become healthy without a premature restart.

## 16. Cleanup

After the experiments, both diagnostic manifests were used to delete their corresponding Kubernetes resources:

```bash
kubectl delete -f manifests/readiness-test.yaml
kubectl delete -f manifests/startup-test.yaml
```

The main application remained available:

```text
nginx-deployment   5/5   5   5
```

The main Service remained:

```text
nginx-service   ClusterIP   10.96.201.49   80/TCP
```

All five main Service endpoints reported `ready=true`.

The diagnostic manifest files were retained locally to make the experiments reproducible.

## 17. Operational Comparison

| Behavior | Readiness | Liveness | Startup |
|---|---|---|---|
| Tests application state | Yes | Yes | Yes, during startup |
| Determines Pod readiness | Directly | Indirectly through container restart | Gates readiness probing |
| Restarts container after failure threshold | No | Yes | Yes |
| Suppresses liveness until initial success | No | No | Yes |
| Suitable for slow initialization | Not independently | Not independently | Yes |

Health probe thresholds should reflect actual application initialization times, expected response times, and recovery requirements.

## 18. Lessons Learned

1. `Running` does not necessarily mean an application is healthy.
2. Readiness Probe determines whether a Pod is ready to receive Service traffic.
3. Readiness failures do not directly restart containers.
4. Liveness Probe can restart an unhealthy container.
5. A container can restart without creating a new Pod object.
6. EndpointSlice readiness reflects the availability of backend endpoints.
7. Startup Probe protects applications during long initialization phases.
8. Different health checks can require different thresholds.
9. Kubernetes events help verify failures and recovery behavior.
10. Isolated diagnostic Pods allow safe experiments without disturbing the main Deployment.
11. YAML validation should precede configuration deployment.
12. Diagnostic Kubernetes resources should be removed after completing tests.

## 19. Laboratory Result

**Status: Completed — technical experiments**

All three Kubernetes health probe mechanisms were successfully tested.

The experiments demonstrated:

- **Liveness:** Failed HTTP probes triggered a container restart and restored application availability.
- **Readiness:** HTTP 403 caused `Ready=False` and `EndpointSlice ready=false`, without restarting the diagnostic container.
- **Startup:** Five startup probe failures were tolerated during a 25-second initialization delay, and the Pod became Ready without restarting.

All temporary diagnostic Kubernetes resources were removed.

The primary Nginx Deployment remained healthy with five ready replicas.

The Deployment manifest and diagnostic manifests are retained for version control, documentation, and future laboratory exercises.
