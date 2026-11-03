# 10 — Kubernetes RBAC and Service Accounts

**Laboratory:** K8S-020
**Environment:** kind-k8s-lab
**Namespace:** lab-apps
**Status:** Practical tests completed

## 1. Objective

Configure and verify Kubernetes Role-Based Access Control (RBAC) according to the principle of least privilege.

The laboratory covers:

- Inspecting existing Kubernetes identities and permissions.
- Creating a dedicated ServiceAccount.
- Defining namespace-scoped permissions using a Role.
- Granting permissions using a RoleBinding.
- Testing allowed and denied operations.
- Using ServiceAccount tokens for authentication.
- Isolating credentials in a temporary kubeconfig.
- Accessing Kubernetes API from inside a Pod.
- Verifying TLS certificates and HTTP status codes.
- Cleaning up temporary test resources.

## 2. Initial Environment

The laboratory uses the following environment:

| Component | Value |
|---|---|
| Kubernetes cluster | k8s-lab |
| Context | kind-k8s-lab |
| Namespace | lab-apps |
| Administrator identity | kubernetes-admin |
| Administrator group | kubeadm:cluster-admins |
| ServiceAccount | pod-reader |
| Role | pod-reader-role |
| RoleBinding | pod-reader-binding |
| Test Pod | rbac-test |
| Main Deployment | nginx-deployment |

Initial inspection:

```bash
kubectl config current-context
kubectl auth whoami
kubectl get serviceaccounts -n lab-apps
kubectl get roles -n lab-apps
kubectl get rolebindings -n lab-apps
kubectl get serviceaccount default -n lab-apps -o yaml
```

Initially, only the `default` ServiceAccount existed in `lab-apps`. No custom Roles or RoleBindings were present.

The administrator could get, create, and delete Pods.

## 3. Kubernetes RBAC Components

**ServiceAccount** provides an identity intended primarily for workloads running in Kubernetes.

**Role** defines permissions within a namespace.

**RoleBinding** grants permissions defined by a Role or ClusterRole to a specified subject within a namespace.

**ClusterRole** defines reusable permission rules and can include cluster-scoped resources.

A Role alone does not grant permissions. Permissions become effective when the appropriate binding exists.

## 4. Creating the ServiceAccount

Manifest: `manifests/pod-reader-sa.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: lab-apps
  labels:
    app: rbac-lab
```

Commands:

```bash
kubectl apply --dry-run=server -f manifests/pod-reader-sa.yaml
kubectl apply -f manifests/pod-reader-sa.yaml
kubectl get serviceaccounts -n lab-apps
kubectl describe serviceaccount pod-reader -n lab-apps
```

Result: `pod-reader` was created successfully.

## 5. Authorization Before RoleBinding

The ServiceAccount identity was simulated using Kubernetes impersonation:

```bash
SA="system:serviceaccount:lab-apps:pod-reader"

kubectl auth can-i get pods -n lab-apps --as="$SA"
kubectl auth can-i list pods -n lab-apps --as="$SA"
kubectl auth can-i create pods -n lab-apps --as="$SA"
kubectl auth can-i delete pods -n lab-apps --as="$SA"
```

Results:

| Operation | Result |
|---|---|
| get pods | no |
| list pods | no |
| create pods | no |
| delete pods | no |

The ServiceAccount initially had no permissions to manage application Pods.

## 6. Creating the Role

Manifest: `manifests/pod-reader-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: lab-apps
  labels:
    app: rbac-lab
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

Commands:

```bash
kubectl apply --dry-run=server -f manifests/pod-reader-role.yaml
kubectl apply -f manifests/pod-reader-role.yaml
kubectl describe role pod-reader-role -n lab-apps
```

Observed rule:

```text
Resources: pods
Verbs: get list watch
```

Authorization checks continued to return `no` because the Role had not yet been assigned to the ServiceAccount.

## 7. Creating the RoleBinding

Manifest: `manifests/pod-reader-rolebinding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: lab-apps
  labels:
    app: rbac-lab
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: lab-apps
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

Commands:

```bash
kubectl apply --dry-run=server -f manifests/pod-reader-rolebinding.yaml
kubectl apply -f manifests/pod-reader-rolebinding.yaml
kubectl describe rolebinding pod-reader-binding -n lab-apps
```

The RoleBinding successfully connected `pod-reader` with `pod-reader-role`.

## 8. Testing RBAC Permissions

Command:

```bash
SA="system:serviceaccount:lab-apps:pod-reader"

for verb in get list watch create update patch delete; do
  printf "%-8s : " "$verb"
  kubectl auth can-i "$verb" pods \
    -n lab-apps \
    --as="$SA"
done
```

Results:

| Verb | Allowed |
|---|---|
| get | yes |
| list | yes |
| watch | yes |
| create | no |
| update | no |
| patch | no |
| delete | no |

The ServiceAccount received read-only permissions for Pods in `lab-apps`.

## 9. Verifying Namespace and Resource Isolation

Additional tests:

```bash
kubectl auth can-i list deployments -n lab-apps --as="$SA"
kubectl auth can-i get secrets -n lab-apps --as="$SA"
kubectl auth can-i list pods -n default --as="$SA"
kubectl auth can-i list pods -n lab-apps --as="$SA"
```

Results:

| Resource | Namespace | Allowed |
|---|---|---|
| Deployments | lab-apps | no |
| Secrets | lab-apps | no |
| Pods | default | no |
| Pods | lab-apps | yes |

The RoleBinding did not grant access outside its defined namespace or resource scope.

## 10. Testing Real API Requests with Impersonation

Commands:

```bash
kubectl get pods -n lab-apps --as="$SA"
kubectl get pod "$POD" -n lab-apps --as="$SA"
kubectl get secrets -n lab-apps --as="$SA"
kubectl get deployments -n lab-apps --as="$SA"
kubectl get pods -n default --as="$SA"
```

Results:

- Listing Pods in `lab-apps`: successful.
- Reading an individual Nginx Pod: successful.
- Listing Secrets: Forbidden.
- Listing Deployments: Forbidden.
- Listing Pods in `default`: Forbidden.

These requests confirmed RBAC enforcement by the Kubernetes API Server.

## 11. Generating a Short-Lived ServiceAccount Token

Permission check:

```bash
kubectl auth can-i create serviceaccounts/token -n lab-apps
```

Result: `yes`.

A short-lived token was generated:

```bash
SA_TOKEN=$(kubectl create token pod-reader \
  -n lab-apps \
  --duration=10m)
```

The token was stored only in a shell variable and was not committed to Git.

## 12. Troubleshooting: Administrator Certificate Authentication

The initial test:

```bash
kubectl --token="$SA_TOKEN" auth whoami
```

Unexpectedly returned:

```text
Username: kubernetes-admin
Credential: X509SHA256
```

The active kind kubeconfig also contained administrator client certificate credentials.

The test therefore did not establish that the request had authenticated solely using the ServiceAccount token.

The solution was to create a separate temporary kubeconfig without administrator credentials.

## 13. Creating an Isolated Temporary Kubeconfig

The existing cluster connection and CA configuration were retained, while user credentials were replaced with an empty credential entry.

```bash
TOKEN_KUBECONFIG=$(mktemp)
chmod 600 "$TOKEN_KUBECONFIG"

kubectl config view --minify --flatten --raw -o json |
python3 -c '
import json
import sys

config = json.load(sys.stdin)
cluster_name = config["clusters"][0]["name"]

config["users"] = [
    {"name": "pod-reader-test", "user": {}}
]
config["contexts"] = [
    {
        "name": "pod-reader-test",
        "context": {
            "cluster": cluster_name,
            "user": "pod-reader-test",
            "namespace": "lab-apps"
        }
    }
]
config["current-context"] = "pod-reader-test"

json.dump(config, sys.stdout)
' > "$TOKEN_KUBECONFIG"
```

The temporary kubeconfig was checked to confirm it contained no client certificate or client key.

Authentication test:

```bash
kubectl \
  --kubeconfig="$TOKEN_KUBECONFIG" \
  --token="$SA_TOKEN" \
  auth whoami
```

Result:

```text
Username: system:serviceaccount:lab-apps:pod-reader
Credential: JTI
```

This confirmed authentication using the ServiceAccount token.

## 14. Testing Authorization with the Token

Using the isolated kubeconfig, the following tests were performed:

```bash
kubectl --kubeconfig="$TOKEN_KUBECONFIG" \
  --token="$SA_TOKEN" get pods -n lab-apps

kubectl --kubeconfig="$TOKEN_KUBECONFIG" \
  --token="$SA_TOKEN" get secrets -n lab-apps

kubectl --kubeconfig="$TOKEN_KUBECONFIG" \
  --token="$SA_TOKEN" get deployments -n lab-apps

kubectl --kubeconfig="$TOKEN_KUBECONFIG" \
  --token="$SA_TOKEN" get pods -n default
```

Results:

| Operation | Result |
|---|---|
| List Pods in lab-apps | Success |
| List Secrets in lab-apps | Forbidden |
| List Deployments in lab-apps | Forbidden |
| List Pods in default | Forbidden |

Temporary credentials were removed from the local environment:

```bash
unset SA_TOKEN
rm -f -- "$TOKEN_KUBECONFIG"
unset TOKEN_KUBECONFIG
```

Unsetting the variable did not revoke the already issued token; its expiration remained controlled by Kubernetes.

## 15. Creating an RBAC Test Pod

Manifest: `manifests/rbac-test-pod.yaml`

The Pod used:

```yaml
spec:
  serviceAccountName: pod-reader
```

Commands:

```bash
kubectl apply --dry-run=server -f manifests/rbac-test-pod.yaml
kubectl apply -f manifests/rbac-test-pod.yaml

kubectl wait \
  --for=condition=Ready \
  pod/rbac-test \
  -n lab-apps \
  --timeout=120s
```

Observed:

```text
Pod: rbac-test
Status: Running
ServiceAccount: pod-reader
Node: k8s-lab-control-plane
```

## 16. Inspecting Projected ServiceAccount Credentials

Commands:

```bash
kubectl get pod rbac-test \
  -n lab-apps \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'

kubectl exec -n lab-apps rbac-test -- \
  ls -la /var/run/secrets/kubernetes.io/serviceaccount/

kubectl get pod rbac-test \
  -n lab-apps \
  -o jsonpath='{.spec.volumes}{"\n"}'
```

The Pod contained these projected files:

| File | Purpose |
|---|---|
| token | ServiceAccount authentication |
| ca.crt | Kubernetes API TLS verification |
| namespace | Current namespace |

The projected volume included a ServiceAccount token with `expirationSeconds: 3607`.

The token file content was not printed or committed to Git.

## 17. Troubleshooting: BusyBox wget

The Alpine container initially provided BusyBox `wget`.

An HTTPS request using:

```bash
wget --ca-certificate="$CA"
```

failed with:

```text
wget: unrecognized option: ca-certificate=...
```

The failure occurred before the request reached Kubernetes API Server.

The environment was inspected:

```bash
kubectl exec -n lab-apps rbac-test -- \
  /bin/sh -c 'command -v apk; id; cat /etc/alpine-release'
```

Observed:

```text
Alpine Linux: 3.21.9
User: root
Package manager: /sbin/apk
```

The required HTTP client was installed in the temporary container:

```bash
kubectl exec -n lab-apps rbac-test -- \
  apk add --no-cache curl
```

The installation succeeded and provided curl 8.14.1.

TLS verification was retained using `curl --cacert`.

## 18. Accessing Kubernetes API from Inside the Pod

Kubernetes API endpoint discovered from the container:

```text
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
```

The container used its automatically mounted ServiceAccount token and CA certificate.

Example request:

```bash
kubectl exec -n lab-apps rbac-test -- \
  /bin/sh -c '
    API="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
    SA_DIR=/var/run/secrets/kubernetes.io/serviceaccount

    curl --silent --show-error \
      --cacert "$SA_DIR/ca.crt" \
      --header "Authorization: Bearer $(cat "$SA_DIR/token")" \
      --output /dev/null \
      --write-out "HTTP STATUS: %{http_code}\n" \
      "$API/api/v1/namespaces/lab-apps/pods"
  '
```

The token was not printed to the terminal.

## 19. HTTP Authorization Results

Observed responses:

| API Request | HTTP Status | Interpretation |
|---|---|---|
| GET `/api/v1/namespaces/lab-apps/pods` | 200 | Allowed |
| GET `/api/v1/namespaces/lab-apps/secrets` | 403 | Forbidden |
| GET `/apis/apps/v1/namespaces/lab-apps/deployments` | 403 | Forbidden |
| GET `/api/v1/namespaces/default/pods` | 403 | Forbidden |

HTTP 200 confirmed authorized read access.

HTTP 403 confirmed that the authenticated ServiceAccount lacked the requested permissions.

The experiment verified RBAC enforcement using the workload's actual credentials from inside the Kubernetes cluster.

## 20. Cleanup and Verification

The temporary test Pod was deleted:

```bash
kubectl delete pod rbac-test \
  -n lab-apps \
  --wait=true \
  --timeout=120s
```

The ServiceAccount, Role, and RoleBinding were deliberately retained.

Authorization was tested again:

```bash
SA="system:serviceaccount:lab-apps:pod-reader"

for verb in get list watch create update patch delete; do
  printf "%-8s : " "$verb"
  kubectl auth can-i "$verb" pods \
    -n lab-apps \
    --as="$SA"
done
```

Results:

```text
get      : yes
list     : yes
watch    : yes
create   : no
update   : no
patch    : no
delete   : no
```

The main Nginx Deployment remained at five ready replicas.

The test Pod manifest was retained in the repository for future reproduction of the experiment.

## 21. Operational Lessons Learned

1. ServiceAccounts provide identities but do not automatically grant application-level permissions.
2. Roles define permissions; RoleBindings assign them.
3. RBAC should follow the principle of least privilege.
4. Namespace-scoped Roles do not grant permissions in other namespaces.
5. Read-only Pod access does not imply access to Secrets or Deployments.
6. `kubectl auth can-i` is useful for preliminary authorization checks.
7. Real API requests should also be tested to verify enforcement.
8. A client kubeconfig can contain more than one authentication mechanism.
9. Authentication tests should isolate credentials to avoid misleading results.
10. ServiceAccount tokens should never be printed, committed, or stored unnecessarily.
11. Workloads can use projected ServiceAccount tokens and trusted CA certificates.
12. Application container images may lack the required diagnostic tools.
13. TLS certificate verification should not be disabled to bypass tool limitations.
14. HTTP 403 is an expected result when RBAC correctly denies access.
15. Deleting a Pod does not automatically remove its ServiceAccount or RBAC bindings.
16. Unused identities and permissions should be reviewed during security audits.

## 22. Laboratory Result

**Practical tests completed successfully.**

Created a dedicated Kubernetes ServiceAccount with read-only access to Pods in `lab-apps`.

Verified the difference between defining permissions using Role and granting them using RoleBinding.

Confirmed authorized and forbidden operations using impersonation, an isolated short-lived ServiceAccount token, and direct HTTPS requests from a test Pod.

Resolved two practical troubleshooting issues involving administrator X.509 credentials and BusyBox `wget`.

Removed the temporary test Pod while retaining the reusable RBAC resources.

The main Nginx Deployment remained healthy throughout the laboratory.

Documentation review, README update, and Git publication are the remaining repository tasks.
