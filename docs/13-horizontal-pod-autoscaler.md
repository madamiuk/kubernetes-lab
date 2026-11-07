# 13 — Kubernetes Horizontal Pod Autoscaler

**Laboratory:** K8S-023
**Environment:** kind-k8s-lab
**Kubernetes:** v1.37.0
**Date:** 2026-11-07
**Status:** Practical experiments completed

## 1. Objective

The purpose of this laboratory was to configure and test automatic application scaling based on CPU utilization.

The laboratory covered:

- Verifying Metrics Server and Metrics API availability.
- Deploying a dedicated test application.
- Configuring an internal Kubernetes Service.
- Creating a Horizontal Pod Autoscaler using `autoscaling/v2`.
- Generating HTTP load from inside the cluster.
- Observing automatic scale-up.
- Examining HPA status conditions and scaling events.
- Stopping the load and observing automatic scale-down.
- Verifying the final application state.

The existing Nginx Deployment in the `lab-apps` namespace was not modified.

## 2. Laboratory Environment

| Component | Configuration |
|---|---|
| Kubernetes context | kind-k8s-lab |
| Kubernetes version | v1.37.0 |
| Container runtime | containerd |
| Architecture | ARM64 |
| Control plane | k8s-lab-control-plane |
| Cluster type | Single-node kind |
| Metrics Server | v0.9.0 |
| Test namespace | hpa-lab |
| Application Deployment | hpa-demo |
| Application Service | hpa-demo |
| Horizontal Pod Autoscaler | hpa-demo |
| Load generator | hpa-load-generator |

The cluster was running locally using Colima and Docker.

## 3. Initial Environment Verification

Commands:

```bash
kubectl config current-context

kubectl get deployment metrics-server -n kube-system

kubectl get apiservice v1beta1.metrics.k8s.io

kubectl describe node k8s-lab-control-plane \
  | grep -A 12 '^Allocated resources:'

kubectl top nodes

kubectl get hpa -A
```

Observed results:

```text
Context: kind-k8s-lab
Metrics Server: 1/1 Ready
Metrics API: Available=True
Existing HPA resources: None
```

Node resource allocation:

| Resource | Requests | Limits |
|---|---|---|
| CPU | 1300m (65%) | 1250m (62%) |
| Memory | 810Mi (20%) | 980Mi (25%) |

Observed node usage:

```text
CPU:    239m (11%)
Memory: 763Mi (19%)
```

The node had approximately 700m of CPU capacity not reserved by existing resource requests.

The available CPU request capacity was sufficient for a small HPA experiment.

## 4. Laboratory Architecture

```text
                 Kubernetes cluster
                        |
                  Namespace hpa-lab
                        |
         +--------------+--------------+
         |                             |
         v                             v
   HTTP Load Generator          Metrics Server
         |                             |
         v                             v
   Service hpa-demo             metrics.k8s.io
         |                             |
         v                             v
   Deployment hpa-demo <--------- HPA Controller
         |
         v
    Application Pods
      1 to 4 replicas
```

The HPA controller uses CPU metrics to adjust the desired replica count of the Deployment.

The Deployment controller then creates or terminates Pods to match the requested replica count.

## 5. Namespace Creation

Manifest:

`manifests/hpa-namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hpa-lab
  labels:
    app.kubernetes.io/part-of: kubernetes-lab
    lab: k8s-023
```

Validation:

```bash
kubectl apply --dry-run=server \
  -f manifests/hpa-namespace.yaml
```

Deployment:

```bash
kubectl apply -f manifests/hpa-namespace.yaml
```

Verification:

```bash
kubectl get namespace hpa-lab --show-labels

kubectl get all -n hpa-lab
```

Observed result:

```text
hpa-lab   Active
```

The namespace initially contained no application resources.

## 6. Test Application Deployment

Manifest:

`manifests/hpa-deployment.yaml`

The application used the image:

```text
registry.k8s.io/hpa-example
```

Deployment configuration:

| Parameter | Value |
|---|---|
| Initial replicas | 1 |
| CPU request | 100m |
| CPU limit | 300m |
| Memory request | 32Mi |
| Memory limit | 128Mi |
| HTTP port | 80 |

The test application was deployed with:

```bash
kubectl apply --dry-run=server \
  -f manifests/hpa-deployment.yaml

kubectl apply -f manifests/hpa-deployment.yaml
```

Initial rollout monitoring:

```bash
kubectl rollout status deployment/hpa-demo \
  -n hpa-lab \
  --timeout=180s
```

The command timed out:

```text
Waiting for deployment "hpa-demo" rollout to finish:
0 of 1 updated replicas are available...

error: timed out waiting for the condition
```

## 7. Troubleshooting the Rollout Timeout

The Deployment was inspected:

```bash
kubectl describe pods -n hpa-lab \
  -l app=hpa-demo
```

Kubernetes events identified the cause:

```text
Pulling image "registry.k8s.io/hpa-example"

Successfully pulled image "registry.k8s.io/hpa-example"
in 3m24.84s
```

The image size was approximately 164 MB.

The first download took longer than the configured 180-second rollout timeout.

The Pod was eventually started successfully:

```text
State: Running
Ready: True
Restart Count: 0
```

A subsequent check succeeded:

```bash
kubectl rollout status deployment/hpa-demo \
  -n hpa-lab \
  --timeout=30s
```

Output:

```text
deployment "hpa-demo" successfully rolled out
```

**Lesson:** A `kubectl rollout status` timeout does not necessarily mean that the Deployment itself failed. The cluster continues processing the workload after the client-side timeout.

## 8. Initial Application Metrics

Command:

```bash
kubectl top pods -n hpa-lab
```

Observed:

```text
CPU:    1m
Memory: 42Mi
```

With a CPU request of 100m, HPA CPU utilization was:

```text
1m / 100m × 100% = 1%
```

The memory usage exceeded its request but remained below the configured 128Mi memory limit.

The application's logs also contained an Apache `ServerName` warning.

The container started successfully despite that warning.

**Security note:** The demonstration image contains legacy Apache and PHP versions. It was used only for the isolated local laboratory and was not exposed to the Internet.

## 9. ClusterIP Service Configuration

Manifest:

`manifests/hpa-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo
  namespace: hpa-lab
  labels:
    app: hpa-demo
spec:
  type: ClusterIP
  selector:
    app: hpa-demo
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

Commands:

```bash
kubectl apply -f manifests/hpa-service.yaml

kubectl get service hpa-demo -n hpa-lab

kubectl describe service hpa-demo -n hpa-lab

kubectl get endpointslices \
  -n hpa-lab \
  -l kubernetes.io/service-name=hpa-demo \
  -o wide
```

Observed:

| Parameter | Value |
|---|---|
| Service type | ClusterIP |
| Service IP | 10.96.158.211 |
| Port | 80/TCP |
| Endpoint | 10.244.0.55:80 |
| Address family | IPv4 |

The internal Service DNS name was:

```text
hpa-demo.hpa-lab.svc.cluster.local
```

No NodePort, LoadBalancer or Ingress was created.

## 10. HTTP Connectivity Verification

A local port-forward was started:

```bash
kubectl port-forward \
  --address 127.0.0.1 \
  -n hpa-lab \
  service/hpa-demo \
  18080:80
```

A second terminal executed:

```bash
curl -i --max-time 10 \
  http://127.0.0.1:18080/
```

Result:

```http
HTTP/1.1 200 OK
Content-Length: 3
Content-Type: text/html; charset=UTF-8

OK!
```

The application responded successfully.

The port-forward was terminated after testing.

Port-forwarding to a Service establishes a connection to a selected backend Pod; it is not itself proof of traffic passing through the Service ClusterIP.

## 11. Horizontal Pod Autoscaler Configuration

Manifest:

`manifests/hpa-autoscaler.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
  namespace: hpa-lab
  labels:
    app: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 1
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

Commands:

```bash
kubectl apply --dry-run=server \
  -f manifests/hpa-autoscaler.yaml

kubectl apply -f manifests/hpa-autoscaler.yaml

kubectl get hpa -n hpa-lab

kubectl describe hpa hpa-demo -n hpa-lab
```

Initial HPA status:

```text
TARGETS:   cpu: 1%/50%
MINPODS:   1
MAXPODS:   4
REPLICAS:  1
```

Initial conditions:

```text
AbleToScale     True   ReadyForNewScale
ScalingActive   True   ValidMetricFound
ScalingLimited  False  DesiredWithinRange
```

The HPA controller successfully calculated utilization based on CPU requests.

## 12. HPA CPU Utilization

The Deployment requests:

```yaml
resources:
  requests:
    cpu: "100m"
```

The HPA target is:

```yaml
averageUtilization: 50
```

For an individual Pod requesting 100m CPU, a usage of 50m corresponds to 50% utilization.

Illustrative values:

| CPU usage | CPU utilization |
|---|---|
| 1m | 1% |
| 30m | 30% |
| 50m | 50% |
| 80m | 80% |
| 150m | 150% |

A simplified replica recommendation formula is:

```text
desiredReplicas =
  ceil(currentReplicas × currentUtilization / targetUtilization)
```

The actual controller also accounts for metric availability, tolerance, stabilization windows, scaling policies and configured replica boundaries.

HPA utilization is based on CPU requests, not CPU limits.

## 13. HTTP Load Generator

Manifest:

`manifests/hpa-load-generator.yaml`

The generator used:

```text
busybox:1.36
```

It repeatedly sent HTTP requests to:

```text
http://hpa-demo.hpa-lab.svc.cluster.local/
```

The load-generation loop was:

```sh
while true; do
  wget -q -O /dev/null \
    http://hpa-demo.hpa-lab.svc.cluster.local/
done
```

Generator resources:

| Resource | Request | Limit |
|---|---|---|
| CPU | 25m | 100m |
| Memory | 16Mi | 64Mi |

The CPU limit prevented the generator from using unlimited CPU capacity.

Deployment:

```bash
kubectl apply --dry-run=server \
  -f manifests/hpa-load-generator.yaml

kubectl apply -f manifests/hpa-load-generator.yaml

kubectl wait \
  --for=condition=Ready \
  pod/hpa-load-generator \
  -n hpa-lab \
  --timeout=120s
```

Observed:

```text
pod/hpa-load-generator condition met
```

Logs:

```text
K8S-023 HTTP load test started
```

The Pod was Running and had zero restarts.

## 14. Initial HPA Response

Approximately 15 seconds after the generator started, the HPA still reported:

```text
cpu: 1%/50%
REPLICAS: 1
```

This was expected because resource metrics are collected periodically.

The initial observation did not yet reflect the new HTTP load.

## 15. Monitoring Autoscaling

Useful commands:

```bash
kubectl get hpa -n hpa-lab -w
```

```bash
kubectl get pods -n hpa-lab -w
```

```bash
kubectl describe hpa hpa-demo -n hpa-lab
```

```bash
kubectl top pods -n hpa-lab
```

The watch commands were executed separately.

A watch command with multiple resource types was rejected by the installed `kubectl`:

```text
error: you may only specify a single resource type
```

The corrected form used one resource type per watch.

## 16. Automatic Scale-Up

Under HTTP load, the HPA increased the Deployment replica count:

```text
1 → 4
```

Kubernetes recorded:

```text
SuccessfulRescale
New size: 4
reason: cpu resource utilization (percentage of request) above target
```

The Deployment controller created three additional application Pods.

All four application Pods were Running and Ready, with zero restarts.

Observed CPU usage:

| Pod suffix | CPU | Memory |
|---|---|---|
| 4lbzh | 1m | 57Mi |
| 6z4fm | 36m | 69Mi |
| p4g7q | 121m | 79Mi |
| rv94b | 77m | 69Mi |
| **Total** | **235m** | **274Mi** |

CPU usage was not evenly distributed among application Pods.

HPA continued to calculate utilization from the metrics available for the managed Pods.

## 17. Maximum Replica Limit

HPA reported CPU utilization values including:

```text
69%
91%
82%
79%
84%
73%
```

All were above the configured target of 50%.

However:

```yaml
maxReplicas: 4
```

prevented additional scaling beyond four replicas.

The HPA reported:

```text
ScalingLimited  True  TooManyReplicas
```

This condition confirmed that the desired replica recommendation exceeded the configured maximum.

## 18. Stopping the HTTP Load

The generator was removed:

```bash
kubectl delete pod hpa-load-generator \
  -n hpa-lab \
  --wait=true \
  --timeout=120s
```

The namespace event history showed the generator's container being stopped.

No manual change was made to the Deployment's replica count.

## 19. CPU Utilization After Load Removal

After the generator stopped, HPA CPU utilization decreased through readings including:

```text
70%
65%
84%
33%
1%
```

The HPA eventually reported:

```text
cpu: 1%/50%
```

Initially, the Deployment still had four replicas.

This was consistent with the HPA's scale-down stabilization behavior.

## 20. HPA Scale-Down Stabilization

The default HPA scale-down stabilization window is approximately five minutes.

During this period, the controller considers recent replica recommendations to avoid unnecessarily frequent scale-down operations.

After the CPU load decreased, the following condition was observed:

```text
AbleToScale  True  ScaleDownStabilized
```

This showed that recent recommendations influenced the scale-down decision.

The HPA did not immediately reduce the Deployment to one replica.

## 21. Automatic Scale-Down

The HPA subsequently issued two scale-down decisions:

```text
SuccessfulRescale
New size: 3
reason: All metrics below target
```

```text
SuccessfulRescale
New size: 1
reason: All metrics below target
```

The complete observed replica sequence was:

```text
1 → 4 → 3 → 1
```

The scale-down was performed automatically.

No `kubectl scale` command was used.

## 22. Final Application State

Commands:

```bash
kubectl get hpa hpa-demo -n hpa-lab

kubectl get deployment hpa-demo -n hpa-lab

kubectl get pods -n hpa-lab
```

Final results:

```text
HPA TARGETS:       cpu: 1%/50%
HPA REPLICAS:      1
Deployment READY:  1/1
Pod READY:         1/1
Pod STATUS:        Running
Pod RESTARTS:      0
```

The original application Pod remained running:

```text
hpa-demo-7b894f7fb5-p4g7q
```

Final HPA conditions:

```text
AbleToScale     True   ReadyForNewScale
ScalingActive   True   ValidMetricFound
ScalingLimited  False  DesiredWithinRange
ScaledToZero    False  NotScaledToZero
```

The `ScaledToZero=False` condition was consistent with `minReplicas: 1`.

## 23. Final Event Verification

Command:

```bash
kubectl get events -n hpa-lab \
  --field-selector involvedObject.kind=HorizontalPodAutoscaler \
  --sort-by=.lastTimestamp
```

Observed HPA events:

```text
SuccessfulRescale  New size: 4
SuccessfulRescale  New size: 3
SuccessfulRescale  New size: 1
```

Reasons:

```text
cpu resource utilization (percentage of request) above target
All metrics below target
All metrics below target
```

The event history confirmed successful scale-up and scale-down.

## 24. Load Generator Cleanup Verification

Command:

```bash
kubectl get pod hpa-load-generator -n hpa-lab
```

Observed:

```text
Error from server (NotFound):
pods "hpa-load-generator" not found
```

The generator was successfully removed.

The application Deployment, Service and HPA were intentionally retained for future inspection.

## 25. Reproducing the Laboratory

Apply the manifests in dependency order:

```bash
kubectl apply -f manifests/hpa-namespace.yaml

kubectl apply -f manifests/hpa-deployment.yaml

kubectl apply -f manifests/hpa-service.yaml

kubectl apply -f manifests/hpa-autoscaler.yaml
```

Check the application:

```bash
kubectl rollout status deployment/hpa-demo \
  -n hpa-lab \
  --timeout=300s

kubectl get hpa -n hpa-lab
```

Start load generation:

```bash
kubectl apply -f manifests/hpa-load-generator.yaml
```

Monitor:

```bash
kubectl get hpa -n hpa-lab -w
```

Stop the generator after the experiment:

```bash
kubectl delete pod hpa-load-generator \
  -n hpa-lab \
  --ignore-not-found
```

Allow HPA to reduce the replica count automatically.

**Note:** An operational Metrics Server and functioning resource metrics API are prerequisites for this CPU-based HPA test.

## 26. Optional Laboratory Cleanup

To remove all K8S-023 resources, delete the namespace:

```bash
kubectl delete -f manifests/hpa-namespace.yaml
```

Deleting the namespace removes its namespaced application resources, including the Deployment, Service and HPA.

This command was documented but was not executed during the completed laboratory.

The namespace and application resources were retained after the experiment.

## 27. Operational Lessons Learned

1. HPA requires usable resource metrics.
2. Metrics Server provides CPU metrics through the resource metrics API.
3. CPU utilization for HPA is calculated relative to CPU requests.
4. CPU limits are not the utilization denominator.
5. CPU requests are necessary for percentage-based CPU scaling calculations.
6. HPA controls the replica count of a scalable target such as a Deployment.
7. The Deployment controller performs the actual Pod creation and termination.
8. The `autoscaling/v2` API supports resource-based scaling.
9. `minReplicas` and `maxReplicas` constrain the resulting replica count.
10. HPA may increase replica count directly from one to four.
11. HTTP load can generate enough CPU utilization to trigger scale-up.
12. Service ClusterIP enables stable addressing independent of Pod IP changes.
13. Metrics and HPA decisions are updated periodically.
14. Scaling decisions are not instantaneous.
15. Application requests must be sized with the cluster's available capacity in mind.
16. HPA scale-down stabilization reduces oscillation.
17. Scale-down may proceed in several steps.
18. HPA conditions explain whether scaling is active, possible or limited.
19. Kubernetes events provide evidence of controller decisions.
20. A client-side rollout timeout does not prove that the Deployment failed.
21. Container image downloads can take longer than expected.
22. Demonstration images containing legacy software must remain isolated.
23. Load-generating workloads should have resource limits.
24. Load generators must be removed when an experiment finishes.
25. Monitoring and autoscaling should be validated using real workload behavior.

## 28. Laboratory Result

**Practical experiments completed successfully.**

A dedicated namespace and test application were deployed.

An internal ClusterIP Service provided access to the application.

The application responded successfully to an HTTP request.

Horizontal Pod Autoscaler was configured using `autoscaling/v2`.

The HPA CPU target was set to 50% of CPU requests.

A controlled HTTP load generator was deployed.

HPA automatically increased the application replica count from one to four.

The maximum replica count was enforced.

After stopping the load, CPU utilization decreased to approximately 1%.

The HPA automatically reduced the replica count from four to three and then to one.

The generator Pod was removed.

The final HPA and Deployment state was healthy.

The laboratory demonstrated a complete automatic scaling cycle:

**1 → 4 → 3 → 1 replicas**

The remaining repository tasks are README updates, final validation and Git publication.
