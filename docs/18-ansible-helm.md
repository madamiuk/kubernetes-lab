
# 18 — Helm Release Automation with Ansible

**Laboratory:** K8S-028

**Environment:** kind-k8s-lab

**Kubernetes:** v1.37.0

**Ansible Core:** 2.21.4

**Ansible Collection:** kubernetes.core 6.5.0

**Python:** 3.14.8

**Helm:** v4.3.1

**Helm Diff:** v3.15.15

**Chart:** nginx-lab 0.2.0

**Application:** Nginx 1.30.6

**Status:** Practical experiments completed

## 1. Objective

This laboratory extends K8S-024 and K8S-027 by introducing Helm release automation with Ansible.

The objectives were to:

- Manage a Helm release using `kubernetes.core.helm`.
- Deploy an application from a local Helm chart.
- Maintain release-specific Helm values.
- Verify Ansible idempotency.
- Install and configure Helm Diff.
- Preview changes before upgrading a release.
- Perform a controlled Helm upgrade.
- Roll back a release using Ansible.
- Validate rollback prerequisites.
- Prevent accidental repeated rollback operations.
- Reconcile declarative configuration after rollback.
- Verify Kubernetes DNS and HTTP connectivity.

## 2. Laboratory Architecture

The existing single-node kind cluster was reused.

The laboratory introduced an independent Helm release in the existing `ansible-lab` Namespace.

```text
MacBook M2
    |
    +-- Ansible Core 2.21.4
    +-- kubernetes.core 6.5.0
    +-- Helm 4.3.1
    +-- Helm Diff 3.15.15
    |
    v
Kubernetes API
    |
    v
kind-k8s-lab
    |
    +-- Namespace: ansible-lab
          |
          +-- Deployment: nginx-ansible
          |     +-- 3 Pods
          |     +-- Managed by Ansible
          |
          +-- Deployment: nginx-ansible-helm
                +-- 2 Pods
                +-- Managed by Helm
                +-- Release revision: 3
```

The Helm-managed application uses its own Deployment, Service, and ServiceAccount.

The existing application from K8S-027 remains independent.

## 3. Existing Helm Releases

The initial Helm environment was inspected using:

```bash
helm version --short
ansible --version
ansible-galaxy collection list kubernetes.core
helm list -A
kubectl config current-context
```

Existing releases included:

| Release | Namespace | Revision |
|---------|-----------|----------|
| nginx-test | helm-lab | 4 |
| grafana | monitoring | 3 |
| prometheus | monitoring | 6 |

The existing monitoring stack and `nginx-test` release were not modified during this laboratory.

## 4. Local Helm Chart

The laboratory reused the chart:

`charts/nginx-lab/`

Chart metadata:

```yaml
apiVersion: v2
name: nginx-lab
type: application
version: 0.2.0
appVersion: "1.30.6"
```

Initial validation:

```bash
helm lint ./charts/nginx-lab
```

Observed result:

```text
1 chart(s) linted, 0 chart(s) failed
```

The optional chart icon recommendation did not prevent validation.

## 5. Repository Structure

The following files were introduced:

```text
ansible/
├── helm/
│   └── nginx-values.yml
└── playbooks/
    ├── helm-nginx.yml
    └── helm-rollback.yml
```

The existing chart remains in:

```text
charts/nginx-lab/
```

The release-specific configuration is stored separately from the chart's default `values.yaml`.

## 6. Helm Release Configuration

The configuration file is:

`ansible/helm/nginx-values.yml`

Final configuration:

```yaml
---
replicaCount: 2

fullnameOverride: nginx-ansible-helm

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 10m
    memory: 16Mi
  limits:
    cpu: 100m
    memory: 64Mi
```

The `fullnameOverride` setting ensures that the new Helm-managed resources do not conflict with the existing `nginx-ansible` Deployment and Service.

The application uses two replicas after the completed rollback experiment.

## 7. Manifest Rendering and Validation

Before deployment, the chart was validated using the custom values file:

```bash
helm lint ./charts/nginx-lab \
  -f ansible/helm/nginx-values.yml
```

The rendered manifests were inspected using:

```bash
helm template nginx-ansible-helm \
  ./charts/nginx-lab \
  --namespace ansible-lab \
  -f ansible/helm/nginx-values.yml
```

The generated resource names were:

- ServiceAccount: `nginx-ansible-helm`
- Service: `nginx-ansible-helm`
- Deployment: `nginx-ansible-helm`
- Test Pod: `nginx-ansible-helm-test-connection`

The test Pod is defined as a Helm test hook and is not created during a standard installation.

The new release did not collide with the existing Kubernetes resources.

## 8. Helm Release Management with Ansible

The playbook is:

`ansible/playbooks/helm-nginx.yml`

The primary module is:

```yaml
kubernetes.core.helm
```

The module is configured with:

```yaml
release_state: present
wait: true
wait_timeout: 2m
```

It uses the local Helm chart and release-specific values file.

The kubeconfig path and Kubernetes context are explicitly specified.

Syntax validation:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-nginx.yml \
  --syntax-check
```

Check mode:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-nginx.yml \
  --check --diff
```

Before the release existed, check mode reported:

```text
ok=2 changed=1 failed=0
```

No release was installed by check mode.

## 9. Initial Helm Installation

The release was installed using:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-nginx.yml
```

Observed result:

```text
ok=2 changed=1 failed=0
```

Release verification:

```bash
helm status nginx-ansible-helm -n ansible-lab

helm history nginx-ansible-helm -n ansible-lab
```

Observed state:

```text
Release: nginx-ansible-helm
Namespace: ansible-lab
Revision: 1
Status: deployed
Description: Install complete
```

The Nginx Deployment became ready with two replicas.

The Service received ClusterIP `10.96.190.242`.

## 10. Initial Idempotency Test

The installation playbook was executed again without modifying configuration:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-nginx.yml
```

Observed result:

```text
ok=2 changed=0 failed=0
```

Helm history still contained only revision 1.

The Deployment remained at two ready replicas.

Ansible reported a warning that the default idempotency detection may miss certain configuration changes.

The warning recommended installing Helm Diff version 3.4.1 or newer.

## 11. Helm Diff Installation

Helm Diff was installed to improve change detection and preview Helm upgrades.

Initially:

```bash
helm plugin list
helm diff version
```

The plugin was not installed.

The Helm version was:

```text
v4.3.1
```

### GnuPG installation

GnuPG was installed through Homebrew:

```bash
brew install gnupg
```

Verified version:

```text
GnuPG 2.5.24
```

### Signing key import

The Helm Diff maintainer's public key was imported using:

```bash
curl -fsSL https://github.com/databus23.gpg \
  | gpg --import
```

The signing key fingerprint was checked using:

```bash
gpg --list-keys --with-fingerprint EA17A2A206AFF8CD
```

Expected and observed fingerprint:

```text
C564 5EF4 7482 257A 1F80 6D2B EA17 A2A2 06AF F8CD
```

### Helm 4 keyring compatibility

The initial plugin installation failed because Helm attempted to read:

```text
~/.gnupg/pubring.gpg
```

GnuPG used a newer public key storage format.

The public key was exported to a separate legacy-format keyring:

```bash
gpg --export EA17A2A206AFF8CD \
  > ~/.gnupg/helm-diff-pubring.gpg

chmod 600 ~/.gnupg/helm-diff-pubring.gpg
```

The installation command specified the exported keyring:

```bash
helm plugin install \
  https://github.com/databus23/helm-diff/releases/latest/download/helm-diff-macos-arm64.tgz \
  --keyring "$HOME/.gnupg/helm-diff-pubring.gpg"
```

Installed plugin verification:

```bash
helm plugin list
helm diff version
```

Observed result:

```text
diff 3.15.15
```

The plugin became available to Helm.

## 12. Helm Diff Baseline

Before upgrading the application, the installed release was compared with the current chart and values:

```bash
helm diff upgrade nginx-ansible-helm \
  ./charts/nginx-lab \
  --namespace ansible-lab \
  --kube-context kind-k8s-lab \
  --values ansible/helm/nginx-values.yml
```

The command produced no differences.

This established that the declared configuration matched the installed release.

## 13. Controlled Helm Upgrade

The replica count in `ansible/helm/nginx-values.yml` was temporarily changed:

```diff
-replicaCount: 2
+replicaCount: 3
```

Chart validation succeeded:

```bash
helm lint ./charts/nginx-lab \
  -f ansible/helm/nginx-values.yml
```

Helm Diff identified only one change:

```diff
 spec:
-  replicas: 2
+  replicas: 3
```

The upgrade was executed with Ansible:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-nginx.yml
```

Observed result:

```text
ok=2 changed=1 failed=0
```

Helm history:

```text
Revision 1: superseded
Revision 2: deployed
```

The Deployment reached three ready replicas.

The two existing Pods were preserved and Kubernetes created one additional Pod.

The original `nginx-ansible` application remained at three replicas.

After the upgrade, Helm Diff produced no differences.

## 14. Idempotency After Upgrade

The Helm playbook was executed again with the three-replica configuration:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-nginx.yml
```

Observed result:

```text
ok=2 changed=0 failed=0
```

The active Helm revision remained 2.

No additional revision was created.

The Deployment remained ready with three replicas.

## 15. Rollback Preparation

The Helm release values were inspected:

```bash
helm get values nginx-ansible-helm \
  -n ansible-lab \
  --revision 1

helm get values nginx-ansible-helm \
  -n ansible-lab \
  --revision 2
```

Revision 1 contained:

```yaml
replicaCount: 2
```

Revision 2 contained:

```yaml
replicaCount: 3
```

A rollback preview was generated:

```bash
helm diff rollback nginx-ansible-helm 1 \
  --namespace ansible-lab \
  --kube-context kind-k8s-lab \
  --no-color
```

The only detected modification was:

```diff
 spec:
-  replicas: 3
+  replicas: 2
```

## 16. Rollback Automation with Ansible

The rollback playbook is:

`ansible/playbooks/helm-rollback.yml`

It uses `ansible.builtin.command` to invoke Helm.

Its safety parameters were:

```yaml
expected_current_revision: 2
rollback_target_revision: 1
```

The playbook performs four tasks:

1. Retrieve Helm release history in JSON format.
2. Validate release status and expected revision.
3. Execute `helm rollback` with `--wait`.
4. Display the rollback result.

The rollback command is executed with:

```text
helm rollback nginx-ansible-helm 1
```

The command also explicitly specifies the namespace, Kubernetes context, timeout, and wait behavior.

### Check mode

Before the actual rollback:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-rollback.yml \
  --check
```

Observed result:

```text
ok=2 changed=0 failed=0 skipped=2
```

The history and validation tasks executed successfully.

The rollback operation was skipped in check mode.

### Actual rollback

The rollback was executed using:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-rollback.yml
```

Observed result:

```text
ok=4 changed=1 failed=0
```

Helm returned:

```text
Rollback was a success! Happy Helming!
```

The resulting release history contained:

```text
Revision 1: superseded
Revision 2: superseded
Revision 3: deployed (Rollback to 1)
```

The Deployment returned to two ready replicas.

## 17. Declarative Configuration Reconciliation

After rollback, the Helm release contained two replicas, but the local values file still declared three.

This mismatch could cause a subsequent Ansible deployment to upgrade the release again.

To correct it, `ansible/helm/nginx-values.yml` was changed back to:

```yaml
replicaCount: 2
```

The configuration was verified using Helm Diff.

No differences were reported.

The normal Ansible deployment playbook was executed again:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/helm-nginx.yml
```

Observed result:

```text
ok=2 changed=0 failed=0
```

The active revision remained 3.

The application remained ready with two replicas.

## 18. Repeated Rollback Protection

The rollback playbook was executed a second time without changing its expected revision.

The playbook required:

```yaml
expected_current_revision: 2
```

However, the active release revision was already 3.

The validation task rejected the operation:

```text
Rollback prerequisites not satisfied.
```

Observed result:

```text
ok=1 changed=0 failed=1
```

The rollback command was not executed.

Helm history remained unchanged, and no fourth revision was created.

This demonstrated fail-fast protection against a repeated rollback based on an outdated release revision.

The revision check is not a complete concurrency-control mechanism. Additional deployment serialization would be required in a multi-operator CI/CD environment.

## 19. Final Kubernetes Verification

Both Nginx applications were inspected using:

```bash
kubectl get deployment,service,pods \
  -n ansible-lab \
  -o wide
```

Final Deployment state:

| Deployment | Ready |
|------------|-------|
| nginx-ansible | 3/3 |
| nginx-ansible-helm | 2/2 |

Both Services use ClusterIP networking:

| Service | ClusterIP |
|---------|-----------|
| nginx-ansible | 10.96.27.97 |
| nginx-ansible-helm | 10.96.190.242 |

### Kubernetes DNS verification

```bash
kubectl exec \
  -n ansible-lab \
  deployment/nginx-ansible-helm \
  -- nslookup nginx-ansible-helm.ansible-lab.svc.cluster.local
```

Observed Service resolution:

```text
10.96.190.242
```

### Helm application HTTP test

```bash
kubectl exec \
  -n ansible-lab \
  deployment/nginx-ansible-helm \
  -- wget -S --spider \
  http://nginx-ansible-helm.ansible-lab.svc.cluster.local/
```

Observed result:

```text
HTTP/1.1 200 OK
Server: nginx/1.30.6
```

### Original application HTTP test

```bash
kubectl exec \
  -n ansible-lab \
  deployment/nginx-ansible \
  -- wget -S --spider \
  http://nginx-ansible.ansible-lab.svc.cluster.local/
```

Observed result:

```text
HTTP/1.1 200 OK
Server: nginx/1.30.6
```

All five Pods were ready with zero restarts.

Both applications remained available after the Helm rollback experiment.

## 20. Operational Lessons

### Release lifecycle management

Ansible can automate Helm installation and upgrades using `kubernetes.core.helm`.

### Preview before apply

Helm Diff provides a manifest-level comparison before changes are applied.

### Idempotency

Repeated executions with unchanged configuration did not create unnecessary Helm revisions.

### Rollback history

A Helm rollback creates a new revision that reproduces the selected historical configuration.

It does not reset the release revision number.

### Configuration reconciliation

After a rollback, declarative configuration must be updated to match the desired restored state.

Otherwise, automation may reapply the change that was rolled back.

### Fail-fast validation

Checking release history and expected revision before rollback helps prevent accidental repeated operations.

### Isolation

Independent release names and resource labels allowed Helm-managed and Ansible-managed applications to coexist within one Namespace.

### Application verification

Deployment readiness, DNS resolution, and HTTP connectivity were verified after the release lifecycle experiments.

## 21. Result

Laboratory K8S-028 successfully demonstrated:

- Helm release installation through Ansible.
- Declarative Helm values management.
- Helm Diff installation and use.
- Change preview before upgrade.
- Controlled scaling from two to three replicas.
- Helm release history verification.
- Idempotency before and after upgrades.
- Controlled rollback through Ansible.
- Rollback safety validation.
- Configuration reconciliation after rollback.
- Final DNS and HTTP application verification.

The final Helm release is `nginx-ansible-helm`, revision 3, with two ready replicas.

The existing Ansible-managed Nginx Deployment remains operational with three ready replicas.

**Laboratory K8S-028 completed successfully.**
