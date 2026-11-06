# 11 — Kubernetes Resource Requests, Limits and ResourceQuotas

**Laboratory:** K8S-021
**Environment:** kind-k8s-lab
**Namespace:** resource-lab
**Status:** Practical experiments completed

## 1. Objective

Learn to manage Kubernetes CPU and memory resources using container requests, limits, LimitRange and ResourceQuota.

The laboratory demonstrates:

- Inspecting node capacity and resource allocation.
- Understanding requests and limits.
- Creating an isolated namespace.
- Configuring default container resource values.
- Rejecting containers that exceed LimitRange constraints.
- Enforcing aggregate namespace resource quotas.
- Limiting the number of Pods.
- Releasing and reusing allocated quota.
- Cleaning up test workloads without affecting existing applications.

## 2. Laboratory Environment

| Component | Value |
|---|---|
| Kubernetes context | kind-k8s-lab |
| Kubernetes version | v1.37.0 |
| Node | k8s-lab-control-plane |
| Container runtime | containerd 2.3.4 |
| Test namespace | resource-lab |
| Existing application namespace | lab-apps |
| LimitRange | resource-lab-limits |
| ResourceQuota | resource-lab-quota |

The single-node kind cluster was running on an ARM64 environment.

All resource management experiments were isolated from the existing Nginx Deployment.

## 3. Initial Node Resource Inventory

Commands:

```bash
kubectl config current-context
kubectl get nodes -o wide

kubectl describe node k8s-lab-control-plane |
  grep -A 14 '^Capacity:'

kubectl describe node k8s-lab-control-plane |
  grep -A 8 '^Allocatable:'

kubectl describe node k8s-lab-control-plane |
  grep -A 12 '^Allocated resources:'
```

Observed allocatable resources:

| Resource | Value |
|---|---|
| CPU | 2 |
| Memory | 3998420Ki |
| Maximum Pods | 110 |

Initial resource allocation:

| Resource | Requests | Limits |
|---|---|---|
| CPU | 1200m (60%) | 1250m (62%) |
| Memory | 610Mi (15%) | 980Mi (25%) |

These values represent declared resource requests and limits, not actual CPU and memory usage.

## 4. Requests and Limits

A resource request influences Kubernetes scheduling and reserves capacity for the workload.

A resource limit restricts the resources a container can consume.

CPU limits can result in throttling.

Memory limits can result in container termination when memory use exceeds the permitted amount.

ResourceQuota controls aggregate resource allocation at the namespace level.

LimitRange defines default resource values and constraints for individual containers.

## 5. Initial Resource Policy Inspection

Commands:

```bash
kubectl get resourcequota -A
kubectl get limitrange -A
```

Results:

```text
No resources found
```

The cluster initially had no ResourceQuota or LimitRange objects.

## 6. Creating an Isolated Namespace

Manifest: `manifests/resource-lab-namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: resource-lab
  labels:
    app: resource-lab
    purpose: resource-management
```

Commands:

```bash
kubectl apply --dry-run=server \
  -f manifests/resource-lab-namespace.yaml

kubectl apply \
  -f manifests/resource-lab-namespace.yaml

kubectl get namespace resource-lab --show-labels
kubectl describe namespace resource-lab
```

Result:

```text
resource-lab   Active
```

The namespace initially had no Pods, ResourceQuota or LimitRange.

The existing Nginx Deployment remained at five ready replicas.

## 7. Creating LimitRange

Manifest: `manifests/resource-lab-limitrange.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-lab-limits
  namespace: resource-lab
  labels:
    app: resource-lab
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "128Mi"
      defaultRequest:
        cpu: "50m"
        memory: "32Mi"
      min:
        cpu: "10m"
        memory: "16Mi"
      max:
        cpu: "500m"
        memory: "256Mi"
```

Validation and creation:

```bash
kubectl apply --dry-run=server \
  -f manifests/resource-lab-limitrange.yaml

kubectl apply \
  -f manifests/resource-lab-limitrange.yaml
```

Verification:

```bash
kubectl describe limitrange resource-lab-limits \
  -n resource-lab
```

Observed:

| Resource | Minimum | Maximum | Default Request | Default Limit |
|---|---|---|---|---|
| CPU | 10m | 500m | 50m | 200m |
| Memory | 16Mi | 256Mi | 32Mi | 128Mi |

## 8. Testing Automatic Resource Defaults

Manifest: `manifests/resource-default-test.yaml`

The BusyBox Pod deliberately omitted the `resources` section.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-default-test
  namespace: resource-lab
  labels:
    app: resource-lab
    test: default-resources
spec:
  restartPolicy: Never
  containers:
    - name: resource-test
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
      args:
        - |
          echo "K8S-021 resource defaults test started"
          sleep 3600
```

The Pod was created and reached Ready state.

The API Server automatically assigned resources from the LimitRange.

Verification:

```bash
kubectl get pod resource-default-test \
  -n resource-lab \
  -o jsonpath='{.spec.containers[0].resources}{"\n"}'
```

Observed:

```json
{
  "limits": {
    "cpu": "200m",
    "memory": "128Mi"
  },
  "requests": {
    "cpu": "50m",
    "memory": "32Mi"
  }
}
```

## 9. LimitRanger Admission Controller

Pod inspection:

```bash
kubectl describe pod resource-default-test \
  -n resource-lab
```

Observed annotation:

```text
kubernetes.io/limit-ranger:
  LimitRanger plugin set: cpu, memory request for container resource-test;
  cpu, memory limit for container resource-test
```

This confirmed that the LimitRanger admission plugin applied the default resource values.

The Pod was Running, Ready and had zero restarts.

## 10. Pod Quality of Service

Command:

```bash
kubectl get pod resource-default-test \
  -n resource-lab \
  -o jsonpath='{.status.qosClass}{"\n"}'
```

Result:

```text
Burstable
```

The container had resource requests lower than its limits.

Kubernetes QoS classes include:

- Guaranteed.
- Burstable.
- BestEffort.

QoS classification influences workload handling under node resource pressure.

## 11. Testing LimitRange Rejection

Manifest: `manifests/resource-limit-exceeded.yaml`

The test Pod requested a memory limit of `512Mi`.

The LimitRange allowed a maximum of `256Mi`.

Test:

```bash
kubectl apply --dry-run=server \
  -f manifests/resource-limit-exceeded.yaml
```

Observed:

```text
Error from server (Forbidden):
pods "resource-limit-exceeded" is forbidden:
maximum memory usage per Container is 256Mi,
but limit is 512Mi
```

The API Server rejected the request.

The Pod was not created.

The existing test Pod continued running.

## 12. Creating ResourceQuota

Manifest: `manifests/resource-lab-quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-lab-quota
  namespace: resource-lab
  labels:
    app: resource-lab
spec:
  hard:
    requests.cpu: "300m"
    limits.cpu: "1000m"
    requests.memory: "256Mi"
    limits.memory: "768Mi"
    pods: "4"
```

Commands:

```bash
kubectl apply --dry-run=server \
  -f manifests/resource-lab-quota.yaml

kubectl apply \
  -f manifests/resource-lab-quota.yaml

kubectl describe resourcequota resource-lab-quota \
  -n resource-lab
```

The ResourceQuota was created successfully.

## 13. Initial ResourceQuota Usage

One test Pod existed when the ResourceQuota was created.

Observed usage:

| Resource | Used | Hard |
|---|---|---|
| requests.cpu | 50m | 300m |
| limits.cpu | 200m | 1000m |
| requests.memory | 32Mi | 256Mi |
| limits.memory | 128Mi | 768Mi |
| pods | 1 | 4 |

The existing Pod was included in ResourceQuota accounting.

ResourceQuota tracks declared resource allocations rather than actual CPU or memory consumption.

## 14. Testing CPU Quota Rejection

Manifest: `manifests/resource-quota-exceeded.yaml`

The test Pod used:

```yaml
resources:
  requests:
    cpu: "300m"
    memory: "32Mi"
  limits:
    cpu: "400m"
    memory: "128Mi"
```

The container configuration satisfied LimitRange constraints.

However, the namespace already used `50m` CPU requests.

The proposed total would have been:

```text
50m + 300m = 350m
```

The ResourceQuota maximum was `300m`.

Test:

```bash
kubectl apply --dry-run=server \
  -f manifests/resource-quota-exceeded.yaml
```

Observed error:

```text
Error from server (Forbidden):
exceeded quota: resource-lab-quota,
requested: requests.cpu=300m,
used: requests.cpu=50m,
limited: requests.cpu=300m
```

The Pod was not created, and quota usage remained unchanged.

## 15. Testing Pod Count Quota

Manifest: `manifests/resource-quota-pods.yaml`

Three additional BusyBox Pods were created:

- quota-test-1
- quota-test-2
- quota-test-3

Commands:

```bash
kubectl apply --dry-run=server \
  -f manifests/resource-quota-pods.yaml

kubectl apply \
  -f manifests/resource-quota-pods.yaml
```

All three Pods reached Ready state.

Correct wait command:

```bash
kubectl wait \
  --for=condition=Ready \
  pods \
  -l test=pod-quota \
  -n resource-lab \
  --timeout=120s
```

All four test Pods were running.

## 16. ResourceQuota at Full Pod Capacity

Observed usage:

| Resource | Used | Hard |
|---|---|---|
| pods | 4 | 4 |
| requests.cpu | 200m | 300m |
| limits.cpu | 800m | 1000m |
| requests.memory | 128Mi | 256Mi |
| limits.memory | 512Mi | 768Mi |

The namespace reached its maximum Pod count while still having available CPU and memory allocations.

A new Pod creation request was tested:

```bash
kubectl run quota-test-5 \
  --image=busybox:1.36 \
  --restart=Never \
  -n resource-lab \
  --dry-run=server \
  --command -- /bin/sh -c 'sleep 3600'
```

Observed:

```text
Error from server (Forbidden):
exceeded quota: resource-lab-quota,
requested: pods=1,
used: pods=4,
limited: pods=4
```

The fifth Pod was not created.

## 17. Troubleshooting kubectl wait

The initial command omitted the resource type:

```bash
kubectl wait \
  --for=condition=Ready \
  -l test=pod-quota \
  -n resource-lab \
  --timeout=120s
```

It returned:

```text
error: at least one resource must be specified to use a selector
```

The corrected command specified `pods`:

```bash
kubectl wait \
  --for=condition=Ready \
  pods \
  -l test=pod-quota \
  -n resource-lab \
  --timeout=120s
```

All three selected Pods then reported `condition met`.

## 18. Releasing ResourceQuota

One Pod was deleted:

```bash
kubectl delete pod quota-test-3 \
  -n resource-lab \
  --wait=true \
  --timeout=120s
```

Quota usage decreased:

| Resource | Before | After |
|---|---|---|
| pods | 4 | 3 |
| requests.cpu | 200m | 150m |
| limits.cpu | 800m | 600m |
| requests.memory | 128Mi | 96Mi |
| limits.memory | 512Mi | 384Mi |

Deleting the Pod released its accounted resources.

## 19. Reusing the Released Quota

The previously rejected fifth Pod was created:

```bash
kubectl run quota-test-5 \
  --image=busybox:1.36 \
  --restart=Never \
  -n resource-lab \
  --command -- /bin/sh -c 'sleep 3600'
```

The Pod reached Ready state.

Verification:

```bash
kubectl get pod quota-test-5 \
  -n resource-lab \
  -o jsonpath='{.spec.containers[0].resources}{"\n"}'
```

Observed resource defaults:

```json
{
  "limits": {
    "cpu": "200m",
    "memory": "128Mi"
  },
  "requests": {
    "cpu": "50m",
    "memory": "32Mi"
  }
}
```

Quota usage returned to four Pods.

This demonstrated quota reclamation and reuse.

## 20. Cleaning Up Test Pods

The four remaining test Pods were deleted:

```bash
kubectl delete pod \
  quota-test-1 \
  quota-test-2 \
  quota-test-5 \
  resource-default-test \
  -n resource-lab \
  --wait=true \
  --timeout=120s
```

The namespace contained no Pods afterward.

ResourceQuota usage returned to zero:

| Resource | Used | Hard |
|---|---|---|
| pods | 0 | 4 |
| requests.cpu | 0 | 300m |
| limits.cpu | 0 | 1000m |
| requests.memory | 0 | 256Mi |
| limits.memory | 0 | 768Mi |

The namespace, LimitRange and ResourceQuota were retained for future experiments.

The Nginx Deployment remained at five ready replicas.

## 21. Operational Lessons Learned

1. Requests are used by the scheduler when placing Pods.
2. CPU and memory limits restrict container resource consumption.
3. LimitRange can assign default requests and limits.
4. The LimitRanger admission plugin can reject invalid resource configurations.
5. ResourceQuota enforces aggregate resource allocations within a namespace.
6. ResourceQuota can account for Pods created before a quota was added.
7. ResourceQuota evaluates resource requests and limits, not actual usage metrics.
8. A Pod may be rejected even when the node has spare CPU and memory.
9. Pod count limits operate independently of CPU and memory quota availability.
10. Removing Pods releases their accounted resource allocations.
11. Reclaimed quota can be reused by new Pods.
12. Resource policies can be tested safely in an isolated namespace.
13. Server-side dry-run is useful for controlled admission tests.
14. kubectl label selectors must be combined with an explicit resource type when using `kubectl wait`.
15. Production workloads should not be modified unnecessarily during experiments.

## 22. Laboratory Result

**Practical experiments completed successfully.**

Created the isolated `resource-lab` namespace.

Configured a LimitRange with CPU and memory defaults, minimums and maximums.

Verified automatic resource assignment and Burstable QoS classification.

Demonstrated admission rejection when container memory limits exceeded the permitted maximum.

Configured ResourceQuota for aggregate CPU requests, CPU limits, memory requests, memory limits and Pod counts.

Verified rejection when aggregate CPU requests exceeded the namespace quota.

Verified rejection when attempting to create a fifth Pod in a namespace limited to four Pods.

Demonstrated resource quota reclamation and reuse after Pod deletion.

Removed all temporary Pods and confirmed that resource usage returned to zero.

Retained the resource management policies for further experiments.

The existing Nginx Deployment remained healthy throughout the laboratory.

Documentation review, README update and Git publication remain the final repository housekeeping tasks.
