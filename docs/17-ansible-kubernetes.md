# 17 — Kubernetes Automation with Ansible: Declarative Application Management

**Laboratory:** K8S-027

**Environment:** kind-k8s-lab

**Kubernetes:** v1.37.0

**Ansible Core:** 2.21.4

**Ansible Collection:** kubernetes.core 6.5.0

**Python:** 3.14.8

**Kubernetes Python Client:** 37.0.2

**Helm:** v4.3.1

**Status:** Practical experiments completed

## 1. Objective

This laboratory introduced Kubernetes automation using Ansible and the kubernetes.core collection.

The objectives were to:

- Configure Ansible access to the existing Kubernetes cluster.
- Retrieve Kubernetes resources using the Kubernetes API.
- Create and manage a Namespace declaratively.
- Deploy and scale an Nginx application through Ansible.
- Configure a ClusterIP Service.
- Verify Kubernetes DNS and HTTP connectivity.
- Test idempotency and Ansible check mode.
- Implement automated Deployment readiness verification.
- Centralize application configuration variables.
- Validate deployment parameters before changing Kubernetes resources.

## 2. Laboratory Environment

The control node is an Apple Silicon MacBook running Ansible inside a Python virtual environment.

```text
MacBook M2
    |
    +-- Python 3.14.8
    +-- Ansible Core 2.21.4
    +-- kubernetes.core 6.5.0
    +-- Kubernetes Python Client 37.0.2
    |
    +-- ~/.kube/config
            |
            | context: kind-k8s-lab
            v
       Kubernetes API
            |
            v
       kind-k8s-lab
            |
            +-- ansible-lab Namespace
                    |
                    +-- nginx-ansible Deployment
                    |       |
                    |       +-- 3 Nginx Pods
                    |
                    +-- nginx-ansible Service
                            |
                            +-- ClusterIP
                            +-- EndpointSlice
```

All Ansible Kubernetes operations execute locally on the control node.

SSH access to Kubernetes nodes is not required.

## 3. Python Dependencies

The existing Python virtual environment was checked using:

```bash
python3 --version
ansible --version
ansible-galaxy collection list kubernetes.core

python3 -m pip show kubernetes
python3 -m pip show jsonpatch
python3 -m pip show PyYAML
```

The Kubernetes Python client and jsonpatch packages were initially missing.

They were installed using:

```bash
python3 -m pip install \
  'kubernetes>=24.2.0' \
  jsonpatch
```

Verified versions:

```text
kubernetes 37.0.2
jsonpatch 1.45
PyYAML 6.0.3
```

Dependency verification:

```bash
python3 -m pip check
```

Observed result:

```text
No broken requirements found.
```

## 4. Repository Structure

The laboratory introduced the following files:

```text
ansible/
├── playbooks/
│   ├── cluster-info.yml
│   ├── deploy-nginx.yml
│   ├── namespace.yml
│   ├── nginx-deployment.yml
│   ├── nginx-service.yml
│   ├── validate-config.yml
│   └── verify-nginx.yml
└── vars/
    └── nginx.yml
```

Each playbook manages a specific responsibility.

The deploy-nginx.yml playbook combines validation, resource management, and readiness checks.

## 5. Kubernetes Resource Inspection

The first playbook was:

`ansible/playbooks/cluster-info.yml`

It uses the `kubernetes.core.k8s_info` module to retrieve Kubernetes Nodes, Namespaces, and Deployments.

The playbook specifies:

```yaml
hosts: localhost
connection: local
gather_facts: false
```

Authentication uses the existing kubeconfig file and explicitly selects the `kind-k8s-lab` context.

The Ansible Python interpreter was pinned using:

```yaml
ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

This avoids relying on automatic interpreter discovery.

Execution:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/cluster-info.yml
```

Observed results:

- One Kubernetes control-plane node.
- Kubernetes v1.37.0.
- Debian GNU/Linux 13.
- ARM64 architecture.
- Ten active Namespaces.
- The nginx-deployment application in lab-apps with five ready and available replicas.

Final execution summary:

```text
ok=6 changed=0 failed=0
```

The inspection playbook does not modify Kubernetes resources.

## 6. Namespace Management

The namespace.yml playbook manages the `ansible-lab` Namespace.

The resource is created using `kubernetes.core.k8s` with:

```yaml
state: present
api_version: v1
kind: Namespace
```

The Namespace includes the labels:

```yaml
app.kubernetes.io/managed-by: ansible
environment: lab
```

Syntax validation:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/namespace.yml \
  --syntax-check
```

Check mode:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/namespace.yml \
  --check --diff
```

The initial check mode reported:

```text
changed=1
```

The Namespace did not yet exist.

The actual deployment was then executed:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/namespace.yml
```

Observed result:

```text
changed=1
```

Kubernetes confirmed the Namespace was Active.

A second execution reported:

```text
changed=0
```

This confirmed idempotent Namespace management.

## 7. Nginx Deployment Management

The nginx-deployment.yml playbook creates an Nginx Deployment in the ansible-lab Namespace.

The application uses:

```text
Deployment: nginx-ansible
Image: nginx:stable-alpine
Container port: 80
CPU request: 10m
Memory request: 16Mi
CPU limit: 100m
Memory limit: 64Mi
```

The initial deployment used two replicas.

The manifest was validated using Ansible syntax checking and check mode before deployment.

Execution:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/nginx-deployment.yml
```

Observed result:

```text
changed=1
```

Kubernetes created:

- One Deployment.
- One ReplicaSet.
- Two running Nginx Pods.

Deployment readiness was verified using:

```bash
kubectl rollout status \
  deployment/nginx-ansible \
  -n ansible-lab \
  --timeout=120s
```

The rollout completed successfully.

A second execution of the playbook returned:

```text
changed=0
```

The original Pods and ReplicaSet were preserved.

## 8. Deployment Scaling

The Deployment was scaled from two to three replicas using an Ansible variable override.

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/nginx-deployment.yml \
  -e nginx_replicas=3
```

Observed result:

```text
changed=1
```

Kubernetes created one additional Pod using the existing ReplicaSet.

The final Deployment status was:

```text
READY       3/3
UP-TO-DATE  3
AVAILABLE   3
```

Running the same playbook again with `nginx_replicas=3` reported:

```text
changed=0
```

The experiment confirmed that replica scaling can be performed declaratively without using `kubectl scale`.

## 9. ClusterIP Service Management

The nginx-service.yml playbook creates a Service for the Nginx Deployment.

The Service uses:

```yaml
type: ClusterIP
selector:
  app: nginx-ansible
ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

Check mode reported the expected planned change.

The Service was created using:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/nginx-service.yml
```

Kubernetes assigned:

```text
Service: nginx-ansible
Type: ClusterIP
ClusterIP: 10.96.27.97
Port: 80/TCP
```

The EndpointSlice contained three ready endpoints:

```text
10.244.0.78 ready=true
10.244.0.77 ready=true
10.244.0.79 ready=true
```

The EndpointSlice was inspected using:

```bash
kubectl get endpointslices \
  -n ansible-lab \
  -l kubernetes.io/service-name=nginx-ansible \
  -o wide
```

Repeated execution of the Service playbook returned:

```text
changed=0
```

The Service ClusterIP and endpoint configuration remained unchanged.

## 10. Kubernetes DNS and HTTP Verification

DNS resolution was tested from an existing Nginx Pod.

```bash
kubectl exec -n ansible-lab \
  deployment/nginx-ansible \
  -- nslookup \
  nginx-ansible.ansible-lab.svc.cluster.local
```

The DNS response returned the Service ClusterIP:

```text
10.96.27.97
```

The DNS server was:

```text
10.96.0.10
```

HTTP connectivity was tested using:

```bash
kubectl exec -n ansible-lab \
  deployment/nginx-ansible \
  -- wget -qO- \
  http://nginx-ansible.ansible-lab.svc.cluster.local/
```

The request returned the standard Nginx welcome page.

HTTP status verification:

```bash
kubectl exec -n ansible-lab \
  deployment/nginx-ansible \
  -- wget -S --spider \
  http://nginx-ansible.ansible-lab.svc.cluster.local/
```

Observed response:

```text
HTTP/1.1 200 OK
```

The tests confirmed DNS resolution and HTTP connectivity through the Service.

They did not independently measure traffic distribution across all three Pods.

## 11. Combined Deployment Playbook

The deploy-nginx.yml playbook imports the component playbooks using:

```yaml
ansible.builtin.import_playbook
```

The complete deployment workflow is:

```text
deploy-nginx.yml
    |
    +-- validate-config.yml
    |
    +-- namespace.yml
    |
    +-- nginx-deployment.yml
    |
    +-- nginx-service.yml
    |
    +-- verify-nginx.yml
```

Initially, the combined deployment required:

```bash
-e nginx_replicas=3
```

A shared configuration file was later introduced to remove this requirement.

## 12. Shared Configuration Variables

The configuration is maintained in:

`ansible/vars/nginx.yml`

Final values:

```yaml
---
lab_namespace: ansible-lab
nginx_replicas: 3
nginx_image: nginx:stable-alpine
kubernetes_context: kind-k8s-lab
```

Component playbooks load these values using:

```yaml
vars_files:
  - ../vars/nginx.yml
```

The Namespace, Deployment, Service, verification, and validation playbooks use the shared configuration.

The final combined deployment runs without extra variables:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/deploy-nginx.yml
```

This makes the intended application state explicit and reproducible from repository files.

## 13. Automated Deployment Readiness Verification

The verify-nginx.yml playbook uses `kubernetes.core.k8s_info` to inspect the Nginx Deployment.

Its readiness checks include:

- Deployment existence.
- Controller observed generation.
- Desired number of ready replicas.
- Desired number of updated replicas.
- Desired number of available replicas.

The playbook uses Ansible retry controls:

```yaml
retries: 12
delay: 5
```

A successful execution reported:

```text
Deployment: nginx-ansible
Namespace: ansible-lab
Desired replicas: 3
Ready replicas: 3
Available replicas: 3
Deployment verification: PASSED
```

The read-only verification completed with:

```text
ok=2 changed=0 failed=0
```

This removed the need for a separate manual rollout-status check in the normal deployment workflow.

## 14. Deployment Configuration Validation

The validate-config.yml playbook uses:

```yaml
ansible.builtin.assert
```

The validation checks:

- Namespace configuration is defined and nonempty.
- Kubernetes context is defined and nonempty.
- Container image is defined and nonempty.
- Replica count contains a nonnegative integer.
- Replica count is between 1 and 5.

The upper limit of five replicas is a laboratory resource-management policy.

Validation of the normal configuration succeeded:

```text
Kubernetes deployment configuration validated.
```

A negative test was performed using:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/validate-config.yml \
  -e nginx_replicas=0
```

The test failed as intended:

```text
assertion: nginx_replicas | int >= 1
evaluated_to: false
changed: false
failed=1
```

The same invalid value was then passed to the complete deployment workflow:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/deploy-nginx.yml \
  -e nginx_replicas=0
```

Execution stopped during configuration validation.

The Namespace, Deployment, Service, and readiness tasks were not executed.

Kubernetes inspection confirmed that the application remained unchanged:

```text
nginx-ansible Deployment: 3/3
nginx-ansible Service: 10.96.27.97:80
```

The experiment demonstrated fail-fast configuration validation before resource modification.

## 15. Final Verification

The complete deployment playbook was executed with the shared configuration:

```bash
ansible-playbook \
  -i localhost, \
  ansible/playbooks/deploy-nginx.yml
```

Observed final result:

```text
ok=6 changed=0 unreachable=0 failed=0
```

The workflow successfully completed:

1. Configuration validation.
2. Namespace management.
3. Deployment management.
4. Service management.
5. Deployment readiness inspection.
6. Readiness reporting.

No changes were required because the cluster already matched the declared configuration.

## 16. Operational Observations

### Declarative resource management

Ansible describes the intended state of Kubernetes resources using `state: present`.

The Kubernetes API and controllers maintain the resulting objects.

### Idempotency

Repeated execution with unchanged configuration produces `changed=0`.

The Namespace, Deployment, and Service were independently tested for idempotency.

### Check mode

The `--check --diff` options were used to evaluate expected changes before resource creation.

Check mode did not create the tested Namespace, Deployment, or Service.

### Configuration management

Centralized variables avoid repeatedly passing application settings through command-line overrides.

The repository now defines three replicas as the expected application state.

### Deployment validation

Configuration validation executes before operations that can modify Kubernetes resources.

An invalid replica count stops the workflow before resource changes.

### Kubernetes API access

Ansible interacts directly with Kubernetes through the Python client.

SSH access to Kubernetes nodes is not required.

### Resource constraints

The laboratory uses a single-node kind cluster.

Small per-container resource requests and limits were selected to minimize resource consumption.

## 17. Result

The laboratory successfully demonstrated:

- Kubernetes resource inspection using Ansible.
- Local Kubernetes API access.
- Declarative Namespace, Deployment, and Service management.
- Ansible check mode and idempotency.
- Deployment scaling through configuration variables.
- EndpointSlice, DNS, and HTTP verification.
- Automated readiness checking.
- Shared, Git-managed configuration variables.
- Fail-fast validation of invalid deployment parameters.
- A complete, repeatable Kubernetes application deployment workflow.

The final Nginx Deployment contains three ready replicas and is accessible through a ClusterIP Service.

The final combined Ansible playbook completes successfully without changes when the Kubernetes cluster matches the declared configuration.

**Laboratory K8S-027 completed successfully.**
