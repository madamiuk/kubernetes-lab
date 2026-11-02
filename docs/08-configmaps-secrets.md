# 08 — Kubernetes ConfigMaps and Secrets

## 1. Objective

Learn how Kubernetes separates application configuration from container images using ConfigMaps and Secrets.

The laboratory demonstrates:

- Delivering HTML content to Nginx using a ConfigMap.
- Updating mounted ConfigMap data without restarting Pods.
- Providing Secret values through environment variables.
- Providing Secret values through mounted files.
- Comparing how environment variables and mounted volumes react to Secret updates.
- Safely cleaning up diagnostic resources.

**Laboratory:** K8S-018

## 2. Environment

| Component | Value |
|---|---|
| Cluster | k8s-lab |
| Namespace | lab-apps |
| Platform | kind |
| Application | Nginx |
| Deployment | nginx-deployment |
| Replicas | 5 |
| ConfigMap | nginx-html |
| Laboratory Secret | app-credentials |
| Secret type | Opaque |

The laboratory was performed using a local Kubernetes cluster.

## 3. Initial State

Before the laboratory:

- The Nginx Deployment had five ready replicas.
- No application-specific ConfigMap existed.
- No Secrets existed in the `lab-apps` namespace.
- The Nginx containers had no explicitly configured application volumes or volume mounts.
- Git was synchronized with `origin/main`.

Initial inspection:

```bash
kubectl get configmaps -n lab-apps
kubectl get secrets -n lab-apps
kubectl get deployment nginx-deployment -n lab-apps
```

The existing `kube-root-ca.crt` ConfigMap was a Kubernetes-managed resource and was not modified.

## 4. Creating the Nginx ConfigMap

Manifest:

`manifests/nginx-configmap.yaml`

Resource:

`nginx-html`

The ConfigMap contains an `index.html` key with HTML content.

Initial page version:

`Version: 1.0`

Validation:

```bash
kubectl apply --dry-run=server \
  -f manifests/nginx-configmap.yaml
```

Deployment:

```bash
kubectl apply -f manifests/nginx-configmap.yaml
```

Verification:

```bash
kubectl get configmap nginx-html -n lab-apps
```

The ConfigMap contained one data entry.

## 5. ConfigMap Idempotency

The same manifest was applied again:

```bash
kubectl apply -f manifests/nginx-configmap.yaml
```

Result:

```text
configmap/nginx-html unchanged
```

This confirmed that applying an unchanged manifest did not modify the resource.

## 6. Mounting the ConfigMap in Nginx

The existing Deployment was modified to use a ConfigMap-backed volume.

Container configuration:

```yaml
volumeMounts:
  - name: nginx-html-volume
    mountPath: /usr/share/nginx/html
    readOnly: true
```

Pod configuration:

```yaml
volumes:
  - name: nginx-html-volume
    configMap:
      name: nginx-html
```

The ConfigMap replaced the default HTML directory inside the Nginx containers.

The container image did not need to be rebuilt.

## 7. YAML Troubleshooting

The first manifest validation failed:

```text
yaml: line 61: did not find expected '-' indicator
```

Cause:

The `volumes` section was incorrectly indented inside `volumeMounts`.

Correct YAML hierarchy:

- `volumeMounts` belongs to the container specification.
- `volumes` belongs to the Pod specification.

After correcting indentation:

```bash
kubectl apply --dry-run=server \
  -f manifests/nginx-deployment.yaml
```

Result:

```text
deployment.apps/nginx-deployment configured (server dry run)
```

## 8. Rolling Update

The updated Deployment was applied:

```bash
kubectl apply -f manifests/nginx-deployment.yaml
```

Rollout verification:

```bash
kubectl rollout status deployment/nginx-deployment \
  -n lab-apps \
  --timeout=180s
```

Result:

```text
deployment "nginx-deployment" successfully rolled out
```

New ReplicaSet:

```text
nginx-deployment-b5b76bff5
```

All five new Pods became Ready with zero restarts.

## 9. Verifying the Mounted HTML

A running Pod was selected:

```bash
POD=$(kubectl get pods -n lab-apps \
  -l app=nginx \
  -o jsonpath='{.items[0].metadata.name}')
```

File verification:

```bash
kubectl exec -n lab-apps "$POD" -- \
  cat /usr/share/nginx/html/index.html
```

HTTP verification:

```bash
kubectl exec -n lab-apps "$POD" -- \
  wget -qO- http://127.0.0.1/
```

Both commands returned the custom HTML containing:

```text
Kubernetes Administration Lab
LAB K8S-018 — ConfigMaps
Version: 1.0
```

## 10. Dynamic ConfigMap Update

The local ConfigMap manifest was updated:

```text
Version: 1.0 → Version: 2.0
```

An additional HTML line was added:

```html
<p>Updated dynamically without restarting Nginx Pods.</p>
```

The ConfigMap was reapplied:

```bash
kubectl apply -f manifests/nginx-configmap.yaml
```

The API Server immediately contained version 2.0.

The mounted file initially still contained version 1.0.

After kubelet propagated the update, all five Pods reported version 2.0.

Verification:

```bash
for POD in $(kubectl get pods -n lab-apps \
  -l app=nginx \
  -o jsonpath='{.items[*].metadata.name}')
do
  kubectl exec -n lab-apps "$POD" -- \
    grep 'Version:' /usr/share/nginx/html/index.html
done
```

HTTP verification confirmed version 2.0 in every Pod.

All containers retained zero restarts, and the ReplicaSet remained unchanged.

## 11. ConfigMap Atomic Writer

The mounted directory contained symbolic links:

```text
index.html -> ..data/index.html
..data -> ..2026_11_02_08_54_03.1918327692
```

Kubelet uses versioned directories and an atomic symbolic-link switch to publish updated data.

This mechanism allows updated files to become visible to applications without recreating their Pods.

Applications must still read or reload the changed files to use updated configuration.

## 12. Creating a Laboratory Secret

A diagnostic Secret was created:

```bash
kubectl create secret generic app-credentials \
  --namespace=lab-apps \
  --from-literal=username=labuser \
  --from-literal=password='<LAB_ONLY_PLACEHOLDER>'
```

The placeholder represents a fictional laboratory value, not a real password.

Verification:

```bash
kubectl describe secret app-credentials -n lab-apps
```

The Secret had type `Opaque` and contained two keys:

```text
username
password
```

The initial password used in the experiment was 29 bytes long.

No real credentials were committed to Git.

**Security note:** Using `--from-literal` with real passwords can expose them in shell history and process arguments. Use an approved secret-management workflow in production.

## 13. Secret as Environment Variables

Diagnostic manifest:

`manifests/secret-env-test.yaml`

The Pod references Secret keys using `secretKeyRef`.

Example:

```yaml
env:
  - name: APP_USERNAME
    valueFrom:
      secretKeyRef:
        name: app-credentials
        key: username
  - name: APP_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-credentials
        key: password
```

The Pod was created and became Ready:

```bash
kubectl apply -f manifests/secret-env-test.yaml

kubectl wait \
  --for=condition=Ready \
  pod/secret-env-test \
  -n lab-apps \
  --timeout=120s
```

Both environment variables were present.

Their contents were not printed.

## 14. Updating a Secret Used as Environment Variables

The test Secret password was updated to a different laboratory value.

Password length changed:

```text
29 bytes → 20 bytes
```

Secret metadata confirmed the new length.

However, the running container still reported:

```text
APP_PASSWORD length: 29
```

Its restart count remained zero.

**Result:** Environment variables initialized from Secrets do not automatically change in existing containers.

## 15. Pod Deletion and Recreation

The test Pod was deleted:

```bash
kubectl delete pod secret-env-test -n lab-apps
```

The initial deletion was interrupted with `Ctrl+C`.

A subsequent `kubectl apply` encountered a Pod still being deleted:

```text
Detected changes to resource secret-env-test
which is currently being deleted
```

A readiness check briefly observed the previous Pod, which was then fully removed.

The Pod was successfully recreated:

```bash
kubectl apply -f manifests/secret-env-test.yaml
```

The new Pod received a different IP address:

```text
10.244.0.40
```

Its password environment variable had the updated length:

```text
APP_PASSWORD length: 20
```

The new Pod had zero restarts.

**Lesson:** Interrupting the `kubectl delete` client does not cancel the deletion request. An unmanaged Pod is not recreated automatically after deletion.

## 16. Secret as a Mounted Volume

Diagnostic manifest:

`manifests/secret-volume-test.yaml`

Configuration:

```yaml
volumeMounts:
  - name: credentials-volume
    mountPath: /etc/app-credentials
    readOnly: true

volumes:
  - name: credentials-volume
    secret:
      secretName: app-credentials
      defaultMode: 0400
```

The Pod became Ready.

The mount provided two files:

```text
/etc/app-credentials/username
/etc/app-credentials/password
```

File contents were not displayed.

Initial password file length:

```text
20 bytes
```

The mount included symbolic links:

```text
password -> ..data/password
username -> ..data/username
```

## 17. Dynamic Secret Volume Update

The laboratory Secret was updated again.

Password length changed:

```text
20 bytes → 32 bytes
```

Immediately after updating the Secret:

- API Server stored 32 bytes.
- Existing environment variable retained 20 characters.
- Mounted file initially contained 20 bytes.
- Both diagnostic containers had zero restarts.

After kubelet synchronization:

```text
/etc/app-credentials/password: 32 bytes
```

The volume's symbolic link changed to a new versioned directory:

```text
..data -> ..2026_11_02_09_11_04.4190358019
```

The environment variable still contained 20 characters.

**Result:** Mounted Secret data was updated automatically without restarting the container.

## 18. Comparison of Configuration Delivery

| Mechanism | Delivery | Updates in Running Container |
|---|---|---|
| ConfigMap as Volume | File | Yes, asynchronously |
| Secret as Environment Variable | Environment | No |
| Secret as Volume | File | Yes, asynchronously |

Mounted updates are not automatically consumed by applications that read configuration only during startup.

Volumes mounted with `subPath` do not receive the normal automatic ConfigMap or Secret updates.

## 19. Security Considerations

Kubernetes Secrets are intended for sensitive data, but require appropriate security controls.

Important principles:

1. Base64 encoding is not encryption.
2. Avoid committing real credentials to Git.
3. Restrict Secret access using RBAC.
4. Enable and manage encryption at rest for sensitive Kubernetes API data.
5. Avoid printing secrets in command output and application logs.
6. Consider external secret-management solutions in production.
7. Avoid passing real passwords directly through shell command arguments.
8. Protect backups and cluster administration credentials.
9. Limit application access to only the Secrets it requires.
10. Remove unused diagnostic Secrets after testing.

Secret metadata inspection:

```bash
kubectl describe secret app-credentials -n lab-apps
```

This reveals key names and sizes, but not their values.

## 20. Cleanup

The diagnostic Pods were removed:

```bash
kubectl delete -f manifests/secret-env-test.yaml --wait=true

kubectl delete -f manifests/secret-volume-test.yaml --wait=true
```

Their absence was verified.

References to `app-credentials` were checked in running Pods and in the Nginx Deployment.

No active dependencies were found.

The laboratory Secret was deleted:

```bash
kubectl delete secret app-credentials -n lab-apps
```

The namespace contained no remaining Secrets.

The ConfigMap `nginx-html` was retained.

The primary Nginx Deployment remained healthy:

```text
nginx-deployment  5/5
```

The ConfigMap retained HTML version 2.0.

## 21. Reproducing the Secret Experiments

The diagnostic Pod manifests contain references to a Secret but do not contain passwords.

To reproduce the tests:

1. Create the `app-credentials` Secret in the `lab-apps` namespace using fictional credentials.
2. Validate the diagnostic Pod manifests.
3. Apply `secret-env-test.yaml` and `secret-volume-test.yaml`.
4. Verify the presence of the environment variables and mounted files.
5. Update the Secret using new fictional values.
6. Compare environment variable lengths with mounted file lengths.
7. Delete both diagnostic Pods.
8. Check that no active workload depends on the Secret.
9. Delete the laboratory Secret.

Do not apply all files in `manifests/` blindly: some are diagnostic resources and require manual setup.

## 22. Lessons Learned

- ConfigMaps separate ordinary configuration from container images.
- ConfigMap changes do not automatically require rebuilding Docker images.
- Updating a ConfigMap resource does not by itself trigger a Deployment rollout.
- ConfigMap volume changes propagate asynchronously.
- Secrets can be consumed through environment variables or files.
- Existing environment variables do not update when Secret values change.
- Mounted Secret data can update without restarting containers.
- Kubelet uses atomic volume updates for mounted ConfigMaps and Secrets.
- Applications must support configuration reloading to benefit from updated files.
- Diagnostic manifests should be kept separate from production application configuration.
- Secrets must be managed with appropriate access controls.
- Kubernetes resource cleanup must be verified before closing a laboratory.

## 23. Laboratory Result

**Status: Technical experiments completed**

The laboratory successfully demonstrated ConfigMap and Secret configuration delivery, update behavior, fault diagnosis, and safe diagnostic-resource cleanup.

The main Nginx Deployment remained available with five ready replicas.

The `nginx-html` ConfigMap remained deployed with version 2.0 of the laboratory page.

The diagnostic Pods and laboratory Secret were removed.

Documentation and Git publication complete the remaining laboratory workflow.
