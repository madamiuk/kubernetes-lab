# 06 — Kubernetes Rolling Updates and Rollbacks

## 1. Objective

Perform a controlled Kubernetes Deployment update, observe ReplicaSet transitions, test the application after deployment, and restore the previous configuration using a rollback.

**Related laboratory:** K8S-016.

## 2. Laboratory Environment

| Component | Value |
|---|---|
| Cluster | k8s-lab |
| Namespace | lab-apps |
| Deployment | nginx-deployment |
| Service | nginx-service |
| ClusterIP | 10.96.201.49 |
| Replicas | 5 |
| Update strategy | RollingUpdate |
| Initial image | nginx:stable-alpine |
| Target image | nginx:1.31.8-alpine |
| Initial Nginx version | 1.30.6 |
| Updated Nginx version | 1.31.8 |

## 3. Initial Deployment Inspection

Commands:

```bash
kubectl get deployment nginx-deployment -n lab-apps

kubectl rollout history \
  deployment/nginx-deployment -n lab-apps

kubectl get replicasets -n lab-apps
```

Initial results:

```text
Deployment: nginx-deployment
READY: 5/5
Revision: 1
ReplicaSet: nginx-deployment-699b5c6747
Image: nginx:stable-alpine
```

The application was healthy before the update.

## 4. RollingUpdate Strategy

Command:

```bash
kubectl get deployment nginx-deployment \
  -n lab-apps \
  -o jsonpath='{.spec.strategy}{"\n"}'
```

Observed configuration:

```json
{
  "rollingUpdate": {
    "maxSurge": "25%",
    "maxUnavailable": "25%"
  },
  "type": "RollingUpdate"
}
```

For five desired replicas:

| Parameter | Calculation | Result |
|---|---|---|
| maxSurge | 25% of 5, rounded up | 2 |
| maxUnavailable | 25% of 5, rounded down | 1 |
| Maximum total Pods during rollout | 5 + 2 | 7 |
| Minimum available replicas | 5 - 1 | 4 |

These are rollout constraints, not a guarantee of zero interrupted requests.

## 5. Container Image Verification

The initial container image was:

```text
nginx:stable-alpine
```

All five Pods reported the same image identifier:

```text
docker.io/library/nginx@sha256:0e52bd054126cdc040b1276c0af347aeb02a41b9cfe305d6094767ac30e97949
```

Command:

```bash
kubectl get pods -n lab-apps \
  -l app=nginx \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image,IMAGE_ID:.status.containerStatuses[0].imageID'
```

The original tag was mutable. Recording the image digest provided a precise identifier for the image used before the update.

## 6. ARM64 Image Compatibility

The target image was checked using:

```bash
docker manifest inspect nginx:1.31.8-alpine
```

The registry manifest included a Linux ARM64 image:

```text
OS: linux
Architecture: arm64
Variant: v8
Manifest digest:
sha256:41c8801e327f9c60c47aa700390ea5cf1862a6852bd27d103ba2cac41a7d6ec7
```

The `docker buildx` command was unavailable because the Buildx CLI plugin was not installed.

The standard Docker manifest inspection command provided the required information without installing additional tools.

## 7. Deployment Manifest Modification

File:

```text
manifests/nginx-deployment.yaml
```

Edited using:

```bash
vim manifests/nginx-deployment.yaml
```

Original configuration:

```yaml
image: nginx:stable-alpine
```

Updated configuration:

```yaml
image: nginx:1.31.8-alpine
```

The replica count, container ports, resource requests, and resource limits were preserved.

## 8. Applying the Rolling Update

Commands:

```bash
kubectl apply -f manifests/nginx-deployment.yaml

kubectl rollout status \
  deployment/nginx-deployment \
  -n lab-apps \
  --timeout=180s
```

Observed rollout progress included:

```text
3 out of 5 new replicas have been updated...
4 out of 5 new replicas have been updated...
2 old replicas are pending termination...
1 old replicas are pending termination...
4 of 5 updated replicas are available...
```

Final result:

```text
deployment "nginx-deployment" successfully rolled out
```

The update completed successfully.

## 9. ReplicaSet Transition

After the update:

| ReplicaSet | Desired | Current | Ready |
|---|---:|---:|---:|
| nginx-deployment-699b5c6747 | 0 | 0 | 0 |
| nginx-deployment-78b5fc59b9 | 5 | 5 | 5 |

The older ReplicaSet was retained for revision history.

The new ReplicaSet managed five healthy Pods.

## 10. Deployment Revision History

Command:

```bash
kubectl rollout history \
  deployment/nginx-deployment -n lab-apps
```

Observed revisions after the update:

```text
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Changing the Pod template image created a new Deployment revision.

## 11. Application Verification After Update

Command:

```bash
kubectl exec -n lab-apps \
  deployment/nginx-deployment -- nginx -v
```

Observed result:

```text
nginx version: nginx/1.31.8
```

An internal HTTP test was performed using a temporary BusyBox Pod:

```bash
kubectl run nginx-http-check \
  -n lab-apps \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -i \
  --command -- \
  wget -S -O /dev/null http://nginx-service/
```

Observed HTTP response:

```text
HTTP/1.1 200 OK
Server: nginx/1.31.8
Content-Length: 896
```

The temporary diagnostic Pod was deleted after completing the request.

## 12. Performing the Rollback

Objective: restore the previous Nginx configuration.

Command:

```bash
kubectl rollout undo \
  deployment/nginx-deployment \
  -n lab-apps \
  --to-revision=1
```

Observed result:

```text
deployment.apps/nginx-deployment rolled back
```

The rollback also produced a warning about the `kubectl.kubernetes.io/last-applied-configuration` annotation.

Rollback status was checked using:

```bash
kubectl rollout status \
  deployment/nginx-deployment \
  -n lab-apps \
  --timeout=180s
```

Result:

```text
deployment "nginx-deployment" successfully rolled out
```

## 13. ReplicaSet State After Rollback

| ReplicaSet | Desired | Current | Ready |
|---|---:|---:|---:|
| nginx-deployment-699b5c6747 | 5 | 5 | 5 |
| nginx-deployment-78b5fc59b9 | 0 | 0 | 0 |

The original ReplicaSet was reused, but five new Pod objects were created.

The newly observed Pod IP addresses were:

```text
10.244.0.21
10.244.0.22
10.244.0.23
10.244.0.24
10.244.0.25
```

All five Pods reached the Running and Ready states.

## 14. Revision Number After Rollback

Command:

```bash
kubectl rollout history \
  deployment/nginx-deployment -n lab-apps
```

Observed result:

```text
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

The original configuration became revision 3.

A rollback does not reset the revision counter to its original value.

## 15. Last-Applied-Configuration Warning

The rollback command reported that the Deployment was previously managed using `kubectl apply`.

It warned that rollback does not update the stored last-applied-configuration annotation.

This matters because the annotation participates in client-side apply calculations.

After rollback, the live Deployment used:

```text
nginx:stable-alpine
```

However, the local YAML manifest still specified:

```text
nginx:1.31.8-alpine
```

The cluster and Git working tree were temporarily inconsistent.

## 16. Restoring Declarative Configuration

The original manifest was restored from Git:

```bash
git restore manifests/nginx-deployment.yaml
```

Verification:

```bash
grep -n 'image:' manifests/nginx-deployment.yaml
git status --short
```

Observed image:

```text
image: nginx:stable-alpine
```

Then the desired configuration was reapplied:

```bash
kubectl apply -f manifests/nginx-deployment.yaml
```

Observed result:

```text
deployment.apps/nginx-deployment configured
```

The Deployment successfully rolled out and retained revision 3.

The original ReplicaSet remained active with five ready replicas.

The local Git working tree was clean and synchronized with `origin/main`.

## 17. Final HTTP Verification

A second temporary BusyBox Pod tested the Service after rollback:

```bash
kubectl run nginx-rollback-check \
  -n lab-apps \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -i \
  --command -- \
  wget -S -O /dev/null http://nginx-service/
```

Observed response:

```text
Connecting to nginx-service (10.96.201.49:80)

HTTP/1.1 200 OK
Server: nginx/1.30.6
Content-Length: 896
```

Nginx version was independently verified:

```bash
kubectl exec -n lab-apps \
  deployment/nginx-deployment -- nginx -v
```

Result:

```text
nginx version: nginx/1.30.6
```

Final Deployment status:

```text
nginx-deployment   5/5   5   5
```

A warning about attaching to the short-lived BusyBox container occurred during testing.

The command successfully retrieved the response through the Pod logs, and the temporary Pod was deleted.

## 18. Lessons Learned

1. RollingUpdate gradually replaces Pods rather than requiring simultaneous replacement.
2. `maxSurge` and `maxUnavailable` control rollout capacity and availability constraints.
3. Changes to the Deployment Pod template create new ReplicaSet revisions.
4. Older ReplicaSets can be retained with zero replicas for rollback purposes.
5. `kubectl rollout undo` restores a previous Pod template as a new revision.
6. A rollback may reuse an existing ReplicaSet while creating new Pod objects.
7. A successful rollout should be followed by application-level tests.
8. Short-lived diagnostic Pods may finish before terminal attachment succeeds.
9. Rollback does not automatically restore the YAML manifest or Git history.
10. Declarative configuration must be reconciled after an imperative rollback.
11. Mutable image tags make exact image tracking and reproducibility harder.
12. A successful rollout does not by itself prove zero downtime throughout the deployment.

## 19. Laboratory Result

**Status: Completed**

Successfully updated Nginx from version 1.30.6 to 1.31.8.

Verified all five replicas and tested HTTP through the Kubernetes Service.

Restored the previous application configuration using Kubernetes rollback.

Confirmed that Nginx 1.30.6 was running again, with five healthy replicas and HTTP 200 responses.

Restored consistency between the live Deployment, local YAML manifest, and Git repository.

The laboratory demonstrated practical Deployment lifecycle management, release verification, rollback procedures, and declarative configuration reconciliation.
