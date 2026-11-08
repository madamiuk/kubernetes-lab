
# 14 — Helm: Kubernetes Application Lifecycle Management

**Laboratory:** K8S-024
**Environment:** kind-k8s-lab
**Kubernetes:** v1.37.0
**Helm:** v4.3.1
**Status:** Practical experiments completed

## 1. Objective

The purpose of this lab was to learn how to package, deploy, configure, upgrade, test, and roll back Kubernetes applications using Helm.

The lab covered:

- Installing Helm with Homebrew.
- Creating a custom Helm chart.
- Understanding `Chart.yaml`, `values.yaml`, and Go templates.
- Configuring replicas, container images, and resource requirements.
- Validating charts and rendered Kubernetes manifests.
- Installing and inspecting Helm releases.
- Testing application connectivity.
- Overriding chart values during upgrades.
- Rolling back to a previous release revision.
- Executing Helm test hooks.
- Updating chart metadata and examining Kubernetes rolling updates.
- Understanding Helm release revisions versus Kubernetes Deployment revisions.

## 2. Environment Verification

Before installing Helm, verify the active Kubernetes context and cluster status.

```bash
kubectl config current-context
kubectl get nodes -o wide
```

The active context was `kind-k8s-lab`, with a Ready control-plane node.

Verify Helm installation:

```bash
command -v helm
helm version
```

Helm was initially unavailable.

Install Helm using Homebrew:

```bash
brew install helm
```

Verify the installation:

```bash
helm version
helm env
helm list --all-namespaces
```

**Observed results:**

- Helm version: `v4.3.1`.
- Executable: `/opt/homebrew/bin/helm`.
- No Helm releases were installed initially.
- Helm used the current Kubernetes context.

## 3. Creating the Helm Chart

Create a directory for local charts:

```bash
mkdir -p charts
```

Generate the chart skeleton:

```bash
helm create charts/nginx-lab
```

Helm created the following structure:

```text
charts/nginx-lab/
├── .helmignore
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── hpa.yaml
    ├── httproute.yaml
    ├── ingress.yaml
    ├── NOTES.txt
    ├── service.yaml
    ├── serviceaccount.yaml
    └── tests/
        └── test-connection.yaml
```

### Key files

| File | Purpose |
|---|---|
| `Chart.yaml` | Chart metadata and version information |
| `values.yaml` | Default configuration values |
| `templates/deployment.yaml` | Kubernetes Deployment template |
| `templates/service.yaml` | Kubernetes Service template |
| `templates/serviceaccount.yaml` | ServiceAccount template |
| `templates/_helpers.tpl` | Reusable Helm template helpers |
| `templates/tests/test-connection.yaml` | Helm test hook |

The initial chart used version `0.1.0`, with `appVersion: "1.16.0"`.

## 4. Configuring values.yaml

Edit the chart configuration:

```bash
vim charts/nginx-lab/values.yaml
```

Configure two Nginx replicas:

```yaml
replicaCount: 2
```

Set the container image:

```yaml
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "stable-alpine"
```

Configure resource requests and limits:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "32Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

The following settings were retained:

- Service type: `ClusterIP`.
- Service port: `80`.
- Autoscaling: disabled.
- Ingress: disabled.
- HTTPRoute: disabled.

### 4.1 Troubleshooting incorrect values

The first validation revealed two configuration problems.

**Problem 1 — Empty image tag**

The configuration still contained:

```yaml
tag: ""
```

As a result, the Deployment template used the chart's default application version:

```text
nginx:1.16.0
```

Correcting the image tag to `stable-alpine` resolved the issue.

**Problem 2 — Incorrect resources indentation**

The `resources` section was initially indented by two spaces instead of being defined at the top level.

The chart rendered successfully, but the expected resource requests and limits were missing from the Deployment.

Correct configuration:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "32Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

**Lesson learned:** A successful `helm lint` does not guarantee that configuration values have the intended structure or are correctly referenced by templates.

## 5. Helm Chart Validation

Run the Helm linter:

```bash
helm lint charts/nginx-lab
```

Observed result:

```text
1 chart(s) linted, 0 chart(s) failed
```

Helm also displayed an informational recommendation to add an icon to `Chart.yaml`.

### 5.1 Render Kubernetes manifests

```bash
helm template nginx-test charts/nginx-lab \
  --namespace helm-lab \
  > /tmp/k8s-024-rendered.yaml
```

Verify the container image:

```bash
grep 'image:' /tmp/k8s-024-rendered.yaml
```

Observed result:

```text
image: "nginx:stable-alpine"
image: busybox
```

The BusyBox image belonged to the Helm test hook.

Verify replica configuration:

```bash
grep 'replicas:' /tmp/k8s-024-rendered.yaml
```

Result:

```text
replicas: 2
```

Verify resource configuration:

```bash
grep -A 8 'resources:' /tmp/k8s-024-rendered.yaml
```

Observed values:

```yaml
resources:
  limits:
    cpu: 200m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 32Mi
```

### 5.2 Render without hooks

Generate application manifests without Helm hooks:

```bash
helm template nginx-test charts/nginx-lab \
  --namespace helm-lab \
  --no-hooks \
  > /tmp/k8s-024-no-hooks.yaml
```

Inspect resource kinds:

```bash
grep '^kind:' /tmp/k8s-024-no-hooks.yaml
```

Result:

```text
kind: ServiceAccount
kind: Service
kind: Deployment
```

### 5.3 Create the namespace

```bash
kubectl create namespace helm-lab \
  --dry-run=client -o yaml \
  | kubectl apply -f -
```

Verify:

```bash
kubectl get namespace helm-lab --show-labels
```

Observed status: `Active`.

### 5.4 Kubernetes API validation

```bash
kubectl apply --dry-run=server \
  -n helm-lab \
  -f /tmp/k8s-024-no-hooks.yaml
```

Observed result:

```text
serviceaccount/nginx-test-nginx-lab created (server dry run)
service/nginx-test-nginx-lab created (server dry run)
deployment.apps/nginx-test-nginx-lab created (server dry run)
```

All three resources passed server-side validation without being created.

Verify that the namespace contains no application resources:

```bash
kubectl get all -n helm-lab
helm list -n helm-lab
```

The namespace was empty, and no Helm releases existed.

## 6. Installing the First Helm Release

Install the chart:

```bash
helm install nginx-test charts/nginx-lab \
  --namespace helm-lab \
  --wait \
  --timeout 5m
```

**Observed result:**

```text
NAME: nginx-test
NAMESPACE: helm-lab
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
```

Inspect the release:

```bash
helm list -n helm-lab
helm status nginx-test -n helm-lab
helm history nginx-test -n helm-lab
```

Inspect Kubernetes resources:

```bash
kubectl get deployments -n helm-lab
kubectl get pods -n helm-lab -o wide
kubectl get services -n helm-lab
kubectl get serviceaccounts -n helm-lab
```

**Observed state:**

- Deployment: `2/2 Ready`.
- Two Nginx Pods: `Running`.
- Pod restarts: `0`.
- Service: `ClusterIP`, `10.96.74.181:80`.
- Chart version: `0.1.0`.
- Release revision: `1`.

Verify the deployed image:

```bash
kubectl get deployment nginx-test-nginx-lab \
  -n helm-lab \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Result:

```text
nginx:stable-alpine
```

Verify resource requirements:

```bash
kubectl get deployment nginx-test-nginx-lab \
  -n helm-lab \
  -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

Observed values:

```json
{"limits":{"cpu":"200m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"32Mi"}}
```

Inspect runtime metrics:

```bash
kubectl top pods -n helm-lab
```

Both Nginx Pods initially consumed approximately `1m` CPU and `3Mi` memory each.

## 7. HTTP Connectivity Test

Forward the Service port to localhost:

```bash
kubectl port-forward \
  --address 127.0.0.1 \
  -n helm-lab \
  service/nginx-test-nginx-lab \
  18081:80
```

In a separate terminal, inspect the HTTP headers:

```bash
curl -I \
  --max-time 10 \
  http://127.0.0.1:18081/
```

Observed response:

```text
HTTP/1.1 200 OK
Server: nginx/1.30.6
```

Verify the status code:

```bash
curl -s -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  --max-time 10 \
  http://127.0.0.1:18081/
```

Result:

```text
HTTP 200
```

Stop the port-forward process with `Ctrl+C`.

## 8. Understanding Helm Values

Inspect user-supplied values:

```bash
helm get values nginx-test -n helm-lab
```

Initial result:

```text
USER-SUPPLIED VALUES:
null
```

Inspect computed values:

```bash
helm get values nginx-test -n helm-lab --all
```

The computed configuration contained:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: stable-alpine
```

**Explanation:** The release was installed without `--set` or an additional values file. Helm used the default values embedded in the chart.

## 9. Helm Upgrade — Scaling from Two to Three Replicas

Upgrade the existing release:

```bash
helm upgrade nginx-test charts/nginx-lab \
  --namespace helm-lab \
  --set replicaCount=3 \
  --wait \
  --timeout 5m
```

Observed result:

```text
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
```

Verify the Deployment:

```bash
kubectl get deployment nginx-test-nginx-lab -n helm-lab
```

Result:

```text
nginx-test-nginx-lab   3/3
```

Inspect release history:

```bash
helm history nginx-test -n helm-lab
```

The first revision became `superseded`, and revision 2 became `deployed`.

Verify supplied values:

```bash
helm get values nginx-test -n helm-lab
```

Result:

```yaml
replicaCount: 3
```

Compare the local chart configuration:

```bash
grep '^replicaCount:' charts/nginx-lab/values.yaml
```

Result:

```yaml
replicaCount: 2
```

**Lesson learned:** Helm stores release-specific value overrides independently of the local `values.yaml` file.

The upgrade scaled the existing ReplicaSet without replacing the two original Pods.

## 10. Helm Rollback

Restore the configuration from release revision 1:

```bash
helm rollback nginx-test 1 \
  --namespace helm-lab \
  --wait \
  --timeout 5m
```

Observed result:

```text
Rollback was a success! Happy Helming!
```

Inspect the history:

```bash
helm history nginx-test -n helm-lab
```

Observed revisions:

| Revision | Status | Description |
|---|---|---|
| 1 | superseded | Install complete |
| 2 | superseded | Upgrade complete |
| 3 | deployed | Rollback to 1 |

The Deployment returned to `2/2 Ready`.

Inspect the values:

```bash
helm get values nginx-test -n helm-lab

helm get values nginx-test -n helm-lab --all \
  | grep '^replicaCount:'
```

Results:

```text
USER-SUPPLIED VALUES:
null
```

```text
replicaCount: 2
```

**Lesson learned:** Helm rollback creates a new release revision and restores the configuration recorded in the selected historical revision.

## 11. Helm Test Hooks

Inspect the test template:

```bash
cat charts/nginx-lab/templates/tests/test-connection.yaml
```

The generated chart defines a Kubernetes Pod with the following annotation:

```yaml
annotations:
  "helm.sh/hook": test
```

The test runs BusyBox and uses `wget` to access the Nginx Service.

Execute the test:

```bash
helm test nginx-test \
  --namespace helm-lab \
  --logs \
  --timeout 120s
```

Observed result:

```text
Phase: Succeeded
```

Test logs:

```text
Connecting to nginx-test-nginx-lab:80 (10.96.74.181:80)
saving to 'index.html'
'index.html' saved
```

The downloaded page was 896 bytes.

Inspect the test Pod:

```bash
kubectl get pods -n helm-lab \
  -l app.kubernetes.io/instance=nginx-test
```

The test Pod entered the `Completed` state.

**Lesson learned:** A successful test Pod does not need to remain running. `helm test` does not create a new release revision.

## 12. Chart Version and Application Version

Edit the chart metadata:

```bash
vim charts/nginx-lab/Chart.yaml
```

Updated metadata:

```yaml
apiVersion: v2
name: nginx-lab
description: Nginx Helm chart for Kubernetes training laboratory
type: application
version: 0.2.0
appVersion: "1.30.6"
```

The application version reflected the Nginx version observed during the HTTP test.

Validate:

```bash
helm lint charts/nginx-lab
helm show chart charts/nginx-lab
```

Render the updated manifests:

```bash
helm template nginx-test charts/nginx-lab \
  --namespace helm-lab \
  --no-hooks \
  > /tmp/k8s-024-v020.yaml
```

Verify the chart label:

```bash
grep 'helm.sh/chart:' /tmp/k8s-024-v020.yaml
```

Result:

```text
helm.sh/chart: nginx-lab-0.2.0
```

Verify the image:

```bash
grep 'image:' /tmp/k8s-024-v020.yaml
```

Result:

```text
image: "nginx:stable-alpine"
```

At this stage, the installed release still referenced chart `0.1.0`. Editing the local chart did not automatically update the cluster.

## 13. Upgrading to Chart 0.2.0

Deploy the updated chart:

```bash
helm upgrade nginx-test charts/nginx-lab \
  --namespace helm-lab \
  --wait \
  --timeout 5m
```

Observed result:

```text
STATUS: deployed
REVISION: 4
DESCRIPTION: Upgrade complete
```

Inspect the release:

```bash
helm list -n helm-lab
helm history nginx-test -n helm-lab
```

The installed release now used:

- Chart: `nginx-lab-0.2.0`.
- App version: `1.30.6`.
- Release revision: `4`.
- Deployment: `2/2 Ready`.

### 13.1 Rolling update triggered by metadata

Inspect ReplicaSets:

```bash
kubectl get replicasets -n helm-lab
```

Observed state:

```text
nginx-test-nginx-lab-65dfb6978b   2   2   2
nginx-test-nginx-lab-7c57c8687    0   0   0
```

The upgrade created a new ReplicaSet even though the image remained unchanged.

Verify the image:

```bash
kubectl get deployment nginx-test-nginx-lab \
  -n helm-lab \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Result:

```text
nginx:stable-alpine
```

Verify the application version label:

```bash
kubectl get deployment nginx-test-nginx-lab \
  -n helm-lab \
  -o jsonpath='{.spec.template.metadata.labels.app\.kubernetes\.io/version}{"\n"}'
```

Result:

```text
1.30.6
```

Inspect Deployment rollout history:

```bash
kubectl rollout history deployment/nginx-test-nginx-lab \
  -n helm-lab
```

Observed Deployment revisions: `1` and `2`.

**Lesson learned:** Updating chart metadata can modify the Deployment's Pod template labels and trigger a rolling update, even when the container image is unchanged.

Helm revision numbers and Kubernetes Deployment rollout revision numbers are independent.

## 14. Final Validation

Delete the previous completed test Pod:

```bash
kubectl delete pod nginx-test-nginx-lab-test-connection \
  -n helm-lab
```

Run the test again:

```bash
helm test nginx-test \
  --namespace helm-lab \
  --logs \
  --timeout 120s
```

Observed result:

```text
Phase: Succeeded
```

The test successfully downloaded the Nginx index page.

Verify the release:

```bash
helm list -n helm-lab
helm history nginx-test -n helm-lab
kubectl get deployment nginx-test-nginx-lab -n helm-lab
kubectl get pods -n helm-lab
```

Final state:

| Parameter | Value |
|---|---|
| Release | `nginx-test` |
| Namespace | `helm-lab` |
| Release revision | `4` |
| Chart version | `0.2.0` |
| App version | `1.30.6` |
| Release status | `deployed` |
| Deployment | `2/2 Ready` |
| Container image | `nginx:stable-alpine` |
| Helm test | `Succeeded` |

Final resource measurements:

```text
CPU:    1m per Nginx Pod
Memory: 3Mi per Nginx Pod
```

The chart remained uncommitted at the end of validation; Git showed the untracked `charts/` directory.

## 15. Important Technical Considerations

### Mutable container tags

The `stable-alpine` tag is mutable. Its contents may change when the image is pulled again.

For production deployments, prefer an immutable image digest or a controlled image version.

### Chart metadata is not image verification

The `appVersion` field is metadata. Helm does not automatically verify whether it matches the application binary inside the container.

### Changes to Pod template metadata

Labels and annotations within `spec.template` are part of the Deployment's Pod template.

Changes to these fields can trigger a new ReplicaSet and rolling update.

### Helm hook lifecycle

Test hooks run separately from normal application workloads. Completed test Pods may remain in the namespace depending on hook deletion policies.

### Validation layers

| Command | Purpose |
|---|---|
| `helm lint` | Validate chart structure and template processing |
| `helm template` | Render Kubernetes manifests locally |
| `kubectl apply --dry-run=server` | Validate manifests using Kubernetes API mechanisms |
| `helm install` | Create a managed release |
| `helm upgrade` | Update an existing release |
| `helm rollback` | Restore a historical release configuration |
| `helm test` | Execute release validation hooks |
| `helm history` | Inspect Helm release revisions |

## 16. Conclusions

The lab demonstrated the complete Helm application lifecycle:

**Create → Configure → Validate → Install → Test → Upgrade → Rollback → Upgrade → Test**

Key outcomes:

1. Created a custom Helm chart for Nginx.
2. Configured image settings, replica counts, and resource limits.
3. Diagnosed configuration errors that passed basic linting.
4. Validated rendered resources using Kubernetes server-side dry-run.
5. Installed a functioning Helm release.
6. Verified HTTP connectivity and container resource usage.
7. Scaled an application using a release-specific value override.
8. Successfully rolled back to a previous revision.
9. Executed Helm test hooks.
10. Updated chart metadata and observed a rolling update.
11. Distinguished Helm release history from Kubernetes Deployment rollout history.

**Final result:** Helm release `nginx-test`, revision 4, chart `nginx-lab-0.2.0`, deployed successfully with two healthy Nginx replicas and a passing integration test.
