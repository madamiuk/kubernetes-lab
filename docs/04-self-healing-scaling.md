# 04 — Kubernetes Self-Healing and Horizontal Scaling

## 1. Objective

Understand how Kubernetes maintains the desired application state using Deployments and ReplicaSets.

Practice automatic Pod replacement, horizontal scaling, resource management, and workload verification.

**Related laboratories:** K8S-010 and K8S-011.

## 2. Laboratory Environment

| Component | Configuration |
|---|---|
| Cluster | k8s-lab |
| Kubernetes | v1.37.0 |
| Namespace | lab-apps |
| Deployment | nginx-deployment |
| ReplicaSet | nginx-deployment-699b5c6747 |
| Application | Nginx |
| Initial replicas | 2 |
| Final replicas | 5 |
| Node | k8s-lab-control-plane |

The cluster consists of one control-plane node running application workloads.

## 3. Understanding Desired State

Kubernetes uses a declarative management model.

The administrator specifies the desired state, while controllers continuously compare it with the actual state.

For example:

```yaml
spec:
  replicas: 2
```

This configuration declares that two Pod replicas should exist.

The Deployment controller manages the ReplicaSet, and the ReplicaSet controller maintains the required Pod count.

If a managed Pod is deleted, the ReplicaSet creates a replacement.

## 4. Self-Healing Experiment

### Objective

Verify that Kubernetes automatically replaces a deleted Pod managed by a ReplicaSet.

### Step 1 — Watch Pod Events

A separate terminal was used to monitor Pod changes:

```bash
kubectl get pods -n lab-apps -w
```

The `-w` option continuously watches resource changes.

### Step 2 — Delete a Managed Pod

One Nginx Pod was deliberately deleted:

```bash
kubectl delete pod \
  nginx-deployment-699b5c6747-6sr6f \
  -n lab-apps
```

Only the Pod was deleted. The Deployment and ReplicaSet remained intact.

### Step 3 — Verify Recovery

Commands:

```bash
kubectl get deployment nginx-deployment -n lab-apps
kubectl get replicaset -n lab-apps
kubectl get pods -n lab-apps -o wide
```

Kubernetes automatically created a replacement Pod:

```text
Deleted:
nginx-deployment-699b5c6747-6sr6f

Replacement:
nginx-deployment-699b5c6747-29gjz
```

The new Pod was assigned to the existing control-plane node.

### Step 4 — Inspect Events

```bash
kubectl get events -n lab-apps \
  --sort-by=.metadata.creationTimestamp
```

Important events included:

```text
Killing
Stopping container nginx

SuccessfulCreate
Created pod: nginx-deployment-699b5c6747-29gjz

Scheduled
Successfully assigned Pod to k8s-lab-control-plane

Pulled
Container image already present on machine

Created
Container created

Started
Container started
```

The replacement reused the locally available Nginx image.

**Result:** Automatic Pod replacement was confirmed.

## 5. Standalone Pod vs Deployment

Earlier, a standalone Pod named `nginx-test` had been deleted.

Kubernetes did not recreate it because no controller managed the Pod.

The Deployment experiment produced a different result.

| Feature | Standalone Pod | Deployment-managed Pod |
|---|---|---|
| ReplicaSet controller | No | Yes |
| Desired replica count | Not maintained | Maintained |
| Replacement after Pod deletion | No | Yes |
| Declarative scaling | Not directly | Yes |
| Rolling updates | No | Yes |

Kubernetes self-healing replaces failed or deleted workload instances to restore the declared state.

It does not guarantee uninterrupted service or protection against complete node failure.

## 6. Horizontal Scaling

### Objective

Increase the number of Nginx Pods using declarative configuration.

### Step 1 — Modify the Manifest

The Deployment manifest was edited:

```bash
vim manifests/nginx-deployment.yaml
```

The replica count was changed from:

```yaml
replicas: 2
```

to:

```yaml
replicas: 5
```

### Step 2 — Apply the Configuration

```bash
kubectl apply -f manifests/nginx-deployment.yaml
```

Observed result:

```text
deployment.apps/nginx-deployment configured
```

### Step 3 — Verify Rollout

```bash
kubectl rollout status \
  deployment/nginx-deployment \
  -n lab-apps \
  --timeout=120s
```

Observed result:

```text
deployment "nginx-deployment" successfully rolled out
```

### Step 4 — Verify Replica Count

```bash
kubectl get deployment nginx-deployment -n lab-apps
kubectl get replicaset -n lab-apps
kubectl get pods -n lab-apps -o wide
```

Observed Deployment status:

```text
NAME               READY   UP-TO-DATE   AVAILABLE
nginx-deployment   5/5     5            5
```

The ReplicaSet reported five desired, current, and ready Pods.

All five Pods were running with zero restarts.

## 7. Scaling Events

The events confirmed:

```text
ScalingReplicaSet
Scaled up replica set nginx-deployment-699b5c6747 from 2 to 5
```

Three additional Pods were created.

The existing ReplicaSet was reused because only the desired replica count changed.

No new Pod template revision was required.

## 8. Resource Planning

Each Nginx container was configured with:

| Resource | Request | Limit |
|---|---:|---:|
| CPU | 50m | 250m |
| Memory | 64 MiB | 128 MiB |

Total declared resources:

| Resource | 2 replicas | 5 replicas |
|---|---:|---:|
| CPU requests | 100m | 250m |
| CPU limits | 500m | 1250m |
| Memory requests | 128 MiB | 320 MiB |
| Memory limits | 256 MiB | 640 MiB |

These values represent resource requests and limits rather than actual CPU or memory consumption.

All five replicas were scheduled on the same Kubernetes node.

## 9. High Availability Considerations

Multiple application replicas improve workload redundancy but do not automatically provide node-level high availability.

In this laboratory:

- All Pods run on one Kubernetes node.
- All workloads depend on the same Colima VM.
- Failure of the only node makes all five application replicas unavailable.
- Pod replacement cannot restore service while the only node remains unavailable.

Future multi-node laboratories will investigate scheduling, Pod distribution, node failure, and workload availability.

## 10. Useful Administration Commands

Watch Pods:

```bash
kubectl get pods -n lab-apps -w
```

Inspect Deployment state:

```bash
kubectl describe deployment nginx-deployment -n lab-apps
```

Inspect ReplicaSets:

```bash
kubectl get replicasets -n lab-apps
```

Show recent namespace events:

```bash
kubectl get events -n lab-apps \
  --sort-by=.metadata.creationTimestamp
```

Check rollout status:

```bash
kubectl rollout status \
  deployment/nginx-deployment -n lab-apps
```

Review rollout history:

```bash
kubectl rollout history \
  deployment/nginx-deployment -n lab-apps
```

## 11. Lessons Learned

1. Kubernetes controllers reconcile actual and desired workload state.
2. ReplicaSets maintain the declared number of Pods.
3. Deleting a managed Pod triggers the creation of a replacement.
4. Replacement Pods are new Kubernetes objects.
5. Scaling can be managed declaratively through YAML.
6. Changing only the replica count does not require a new ReplicaSet.
7. Kubernetes resource requests influence scheduling decisions.
8. Multiple Pods on a single node do not provide node-level high availability.
9. Kubernetes events are essential for understanding workload lifecycle changes.

## 12. Laboratory Result

**Status: Completed**

The self-healing experiment confirmed that ReplicaSet automatically replaced a deleted Nginx Pod.

The Deployment was subsequently scaled from two to five replicas using a declarative YAML manifest.

All five Pods reached the Running and Ready states.

The experiments demonstrated desired-state reconciliation, automatic Pod replacement, and horizontal workload scaling.
