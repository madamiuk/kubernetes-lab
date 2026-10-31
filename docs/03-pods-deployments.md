# 03 — Kubernetes Pods and Deployments

## 1. Objective

Understand the differences between standalone Pods and controller-managed workloads, deploy an Nginx application, inspect its lifecycle, and introduce declarative Kubernetes configuration.

**Related laboratories:** K8S-007, K8S-008 and K8S-009.

## 2. Kubernetes Workload Concepts

A Pod is the smallest deployable computing unit in Kubernetes.

A Pod can contain one or more containers that share networking and certain storage resources.

A Deployment manages ReplicaSets, which maintain the desired number of Pods.

```text
Deployment
    |
    +-- ReplicaSet
        |
        +-- Pod
        |    +-- Container
        |
        +-- Pod
             +-- Container
```

Standalone Pods are useful for experiments and troubleshooting, but Deployments are generally more appropriate for stateless applications requiring automated replacement and updates.

## 3. Application Namespace

The namespace `lab-apps` was created to separate application workloads from Kubernetes system components.

Command:

```bash
kubectl create namespace lab-apps
```

Verification:

```bash
kubectl get namespaces
```

The namespace was subsequently used for the Nginx application and diagnostic workloads.

## 4. Creating the First Nginx Pod

Command:

```bash
kubectl run nginx-test \
  --image=nginx:stable-alpine \
  --namespace=lab-apps \
  --port=80
```

Observed result:

```text
pod/nginx-test created
```

The command created a standalone Pod without a Deployment or ReplicaSet.

## 5. Pod Lifecycle

Initial inspection:

```bash
kubectl get pods -n lab-apps -o wide
```

The Pod initially reported:

```text
READY   STATUS
0/1     ContainerCreating
```

The image was not initially present on the node.

Detailed inspection:

```bash
kubectl describe pod nginx-test -n lab-apps
```

The Kubernetes events showed the following sequence:

1. Scheduled — the Pod was assigned to the control-plane node.
2. Pulling — the Nginx image download started.
3. Pulled — the image was successfully downloaded.
4. Created — the container was created.
5. Started — the container began running.

The initial image download took approximately 19.6 seconds.

The Pod eventually reached:

```text
State: Running
Ready: True
Restart Count: 0
```

Its observed IP address was:

```text
10.244.0.5
```

## 6. Container Logs

Command:

```bash
kubectl logs nginx-test -n lab-apps
```

The logs confirmed that Nginx started successfully.

Observed details included:

```text
nginx/1.30.6
using the "epoll" event method
Configuration complete; ready for start up
```

The entrypoint also enabled IPv6 listening in the default Nginx configuration.

This did not expose the application publicly.

## 7. Pod Quality of Service

The standalone Nginx Pod had no CPU or memory requests or limits.

Kubernetes assigned it:

```text
QoS Class: BestEffort
```

A BestEffort Pod is particularly vulnerable to eviction during node resource pressure.

Resource management was introduced later in the Deployment manifest.

## 8. HTTP Testing with Port Forwarding

Command:

```bash
kubectl port-forward \
  -n lab-apps \
  pod/nginx-test \
  8080:80
```

The local port 8080 was forwarded to port 80 in the Pod.

HTTP verification:

```bash
curl -I http://127.0.0.1:8080/
```

Observed response:

```text
HTTP/1.1 200 OK
Server: nginx/1.30.6
Content-Type: text/html
```

This confirmed that the Nginx server was functioning.

Port forwarding provided temporary diagnostic access rather than persistent application exposure.

## 9. Troubleshooting — Missing favicon.ico

A browser request produced the following log message:

```text
GET /favicon.ico HTTP/1.1
404 Not Found
```

Nginx attempted to access:

```text
/usr/share/nginx/html/favicon.ico
```

The file was not present in the default container image.

The HTTP 404 response was expected behavior for a missing resource and did not indicate a Kubernetes or Nginx service failure.

The application root URL returned HTTP 200.

## 10. Deleting the Standalone Pod

Before deletion, the Pod was confirmed as healthy:

```bash
kubectl get pod nginx-test -n lab-apps
```

Observed result:

```text
nginx-test   1/1   Running   0
```

The standalone Pod was then deleted:

```bash
kubectl delete pod nginx-test -n lab-apps
```

Verification:

```bash
kubectl get pods -n lab-apps
```

Observed result:

```text
No resources found in lab-apps namespace.
```

Kubernetes did not recreate the Pod because no workload controller was responsible for maintaining it.

This demonstrated an important distinction between restarting containers in an existing Pod and recreating a deleted Pod object.

## 11. Creating an Nginx Deployment

A declarative manifest was created at:

```text
manifests/nginx-deployment.yaml
```

The initial Deployment configuration used two replicas:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: lab-apps
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "128Mi"
```

The manifest in the repository was subsequently updated to five replicas during the scaling laboratory.

## 12. Manifest Validation and Deployment

Client-side validation:

```bash
kubectl apply \
  --dry-run=client \
  -f manifests/nginx-deployment.yaml
```

Observed result:

```text
deployment.apps/nginx-deployment created (dry run)
```

Deployment:

```bash
kubectl apply -f manifests/nginx-deployment.yaml
```

Observed result:

```text
deployment.apps/nginx-deployment created
```

Rollout verification:

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

## 13. Deployment and ReplicaSet Verification

Commands:

```bash
kubectl get deployments -n lab-apps
kubectl get replicasets -n lab-apps
kubectl get pods -n lab-apps -o wide
```

The initial Deployment created:

```text
Deployment: nginx-deployment
ReplicaSet: nginx-deployment-699b5c6747
Replicas:   2
```

Both Pods reached the Running and Ready states.

The ReplicaSet maintained the declared number of replicas.

## 14. Resource Requests and Limits

Each Nginx container was configured with:

| Resource | Request | Limit |
|---|---|---|
| CPU | 50m | 250m |
| Memory | 64 MiB | 128 MiB |

CPU requests are used by the Scheduler when evaluating placement.

CPU limits restrict available CPU processing time through runtime enforcement.

Memory limits constrain container memory usage and may result in an OOM kill when exceeded.

The Nginx Pods qualify for the Burstable QoS class because their requests and limits are defined but are not equal for both CPU and memory.

## 15. Useful Administration Commands

List application Pods:

```bash
kubectl get pods -n lab-apps
```

Inspect Pod configuration and events:

```bash
kubectl describe pod POD_NAME -n lab-apps
```

Read Pod logs:

```bash
kubectl logs POD_NAME -n lab-apps
```

Inspect Deployments:

```bash
kubectl get deployments -n lab-apps
```

Inspect ReplicaSets:

```bash
kubectl get replicasets -n lab-apps
```

Check rollout:

```bash
kubectl rollout status deployment/nginx-deployment -n lab-apps
```

## 16. Lessons Learned

1. A Pod is the basic execution unit in Kubernetes.
2. A Pod is not equivalent to a Docker container.
3. Pod creation involves scheduling, image retrieval and container startup.
4. ContainerCreating is a normal transitional state.
5. Logs and events provide complementary troubleshooting information.
6. A standalone Pod is not recreated after its API object is deleted unless another controller manages its replacement.
7. A Deployment manages ReplicaSets, which maintain Pod replicas.
8. Resource requests and limits influence scheduling and runtime behavior.
9. Declarative YAML enables repeatable application deployment.

## 17. Laboratory Result

**Status: Completed**

A standalone Nginx Pod was created, inspected, accessed over HTTP and deleted.

A declarative Nginx Deployment was then created and successfully rolled out with two replicas.

Further experiments with Pod replacement and scaling are documented in `04-self-healing-scaling.md`.
