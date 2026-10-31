# 05 — Kubernetes Services, DNS and Networking

## 1. Objective

Understand Kubernetes Service discovery, internal DNS resolution, EndpointSlices, and network communication between Pods.

Verify how a ClusterIP Service provides a stable access point for multiple Nginx replicas.

**Related laboratories:** K8S-012, K8S-013 and K8S-014.

## 2. Laboratory Environment

| Component | Configuration |
|---|---|
| Cluster | k8s-lab |
| Kubernetes | v1.37.0 |
| Namespace | lab-apps |
| Deployment | nginx-deployment |
| Replicas | 5 |
| Service | nginx-service |
| Service type | ClusterIP |
| ClusterIP observed | 10.96.201.49 |
| Service port | 80/TCP |
| Client Pod | dns-client |
| Client image | busybox:1.36 |
| DNS server | 10.96.0.10 |

All application Pods were running on the single `k8s-lab-control-plane` node.

## 3. Kubernetes Networking Overview

The local cluster uses separate address ranges for nodes, Pods, and Services.

| Network | Purpose |
|---|---|
| 172.18.0.0/16 | Docker network containing the kind node |
| 10.244.0.0/24 | Pod subnet assigned to the node |
| 10.96.0.0/16 | Kubernetes Service address range |

The Nginx application uses five Pods with individual IP addresses.

A Kubernetes Service provides a stable virtual IP address, allowing clients to connect without knowing the individual Pod addresses.

## 4. Creating the Nginx Service

The following manifest was saved as:

`manifests/nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: lab-apps
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

The selector `app: nginx` matches the labels of Pods created by the Nginx Deployment.

### Validation

```bash
kubectl apply --dry-run=client \
  -f manifests/nginx-service.yaml
```

### Deployment

```bash
kubectl apply \
  -f manifests/nginx-service.yaml
```

## 5. Service Verification

Commands:

```bash
kubectl get svc -n lab-apps -o wide

kubectl describe svc nginx-service -n lab-apps
```

Observed configuration:

```text
Name:              nginx-service
Namespace:         lab-apps
Selector:          app=nginx
Type:              ClusterIP
IP:                10.96.201.49
Port:              80/TCP
TargetPort:        80/TCP
Session Affinity:  None
```

The Service was configured for IPv4 communication.

Its ClusterIP was dynamically assigned by Kubernetes.

## 6. EndpointSlice Discovery

Kubernetes uses EndpointSlice resources to represent the network endpoints associated with Services.

Command:

```bash
kubectl get endpointslices \
  -n lab-apps \
  -l kubernetes.io/service-name=nginx-service \
  -o wide
```

Observed EndpointSlice:

```text
nginx-service-n2lfs
AddressType: IPv4
Port: 80
```

The EndpointSlice contained five Nginx Pod addresses.

### Endpoint Readiness

```bash
kubectl get endpointslices \
  -n lab-apps \
  -l kubernetes.io/service-name=nginx-service \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{"\n"}{end}'
```

Observed results:

```text
10.244.0.7 ready=true
10.244.0.9 ready=true
10.244.0.10 ready=true
10.244.0.8 ready=true
10.244.0.11 ready=true
```

All five endpoints were reported as ready.

## 7. HTTP Testing through a Service

The Service was first tested using local port forwarding.

Command:

```bash
kubectl port-forward \
  -n lab-apps \
  service/nginx-service \
  8080:80
```

The HTTP test was executed from macOS:

```bash
curl -I http://127.0.0.1:8080/
```

Observed result:

```text
HTTP/1.1 200 OK
Server: nginx/1.30.6
Content-Type: text/html
```

**Important:** `kubectl port-forward service/...` forwards traffic to a selected Pod associated with the Service. It does not demonstrate load balancing across all Service endpoints.

For testing actual Service routing, a client was subsequently deployed inside the Kubernetes cluster.

## 8. Creating a Diagnostic Client Pod

A BusyBox Pod was created:

```bash
kubectl run dns-client \
  -n lab-apps \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Readiness verification:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/dns-client \
  -n lab-apps \
  --timeout=120s
```

Observed result:

```text
pod/dns-client condition met
```

The diagnostic Pod was used for DNS and HTTP tests within the cluster.

## 9. DNS Resolver Configuration

Command:

```bash
kubectl exec -n lab-apps dns-client -- \
  cat /etc/resolv.conf
```

Observed configuration:

```text
search lab-apps.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

The DNS server was the ClusterIP of the CoreDNS Service.

The search domains allowed applications to resolve services using short names within the appropriate namespace.

## 10. Fully Qualified Domain Name Resolution

Command:

```bash
kubectl exec -n lab-apps dns-client -- \
  nslookup nginx-service.lab-apps.svc.cluster.local
```

Observed response:

```text
Server: 10.96.0.10
Address: 10.96.0.10:53

Name: nginx-service.lab-apps.svc.cluster.local
Address: 10.96.201.49
```

The FQDN resolved to the expected Service ClusterIP.

The DNS name follows the format:

`service-name.namespace.svc.cluster.local`

## 11. Short Name Resolution and BusyBox Behavior

Command:

```bash
kubectl exec -n lab-apps dns-client -- \
  nslookup nginx-service
```

The output contained the correct DNS resolution:

```text
Name: nginx-service.lab-apps.svc.cluster.local
Address: 10.96.201.49
```

However, BusyBox also reported NXDOMAIN responses for other search-domain combinations.

The command terminated with exit code 1.

### Analysis

The configured DNS search path caused additional name-resolution attempts.

Some candidate names did not exist and therefore returned NXDOMAIN.

Despite the nonzero exit status, the valid service name was resolved successfully.

A subsequent HTTP test using the same short name also succeeded.

**Lesson:** DNS diagnostics should consider both the resolved addresses and the behavior of the particular DNS client utility.

## 12. Internal HTTP Communication

Command:

```bash
kubectl exec -n lab-apps dns-client -- \
  wget -S -O /dev/null http://nginx-service:80/
```

Observed result:

```text
Connecting to nginx-service:80 (10.96.201.49:80)

HTTP/1.1 200 OK
Server: nginx/1.30.6
```

This confirmed successful HTTP communication from the diagnostic Pod to the Nginx Service.

No macOS port forwarding was required for this request.

The communication followed the logical path:

```text
BusyBox Pod
    |
    +-- DNS query
    |   |
    |   +-- CoreDNS
    |       |
    |       +-- nginx-service -> 10.96.201.49
    |
    +-- HTTP connection to Service ClusterIP
        |
        +-- Kubernetes Service routing
            |
            +-- Selected Nginx Pod
```

## 13. Service Traffic Distribution Experiment

### Objective

Investigate how HTTP connections are distributed across five Nginx replicas.

### Request Generation

Fifteen independent HTTP requests were generated from the BusyBox Pod:

```bash
for i in $(seq 1 15); do
  kubectl exec -n lab-apps dns-client -- \
    wget -q -O /dev/null http://nginx-service/
done
```

The test completed without reported command errors.

### Log Collection

```bash
kubectl logs \
  -n lab-apps \
  -l app=nginx \
  --all-containers=true \
  --prefix=true \
  --since=5m \
  --max-log-requests=10
```

The `--prefix=true` option identified the Pod associated with each log entry.

### Observed Distribution

The logs contained 16 HTTP requests, including one request from the preceding laboratory.

Only the 15 requests generated during the distribution experiment were counted.

| Pod suffix | Pod IP | Requests |
|---|---|---:|
| pvz8h | 10.244.0.7 | 8 |
| z8t66 | 10.244.0.9 | 3 |
| 29gjz | 10.244.0.8 | 2 |
| 8t79j | 10.244.0.11 | 1 |
| r9wqf | 10.244.0.10 | 1 |
| **Total** | | **15** |

All 15 recorded requests returned HTTP 200.

The requests reached all five replicas.

## 14. Traffic Distribution Analysis

The distribution was uneven.

One Pod handled eight requests, while two other Pods handled only one request each.

This does not necessarily indicate a networking problem.

Kubernetes Services generally distribute connections rather than guaranteeing an equal number of HTTP requests per Pod.

The behavior depends on the networking implementation, connection reuse, endpoint selection, and traffic characteristics.

A sample of 15 requests is insufficient for a meaningful statistical assessment of traffic balance.

The experiment nevertheless demonstrated that multiple Nginx replicas were reachable through the same Service address.

## 15. Client IP Observation

The Nginx access logs showed:

```text
10.244.0.12 - - "GET / HTTP/1.1" 200
```

The address `10.244.0.12` belonged to the diagnostic client Pod.

This confirmed that the requests originated from within the Kubernetes Pod network rather than from the macOS host.

## 16. Service Discovery and Workload Changes

Service routing relies on dynamically maintained endpoints.

When Deployment-managed Pods are created, deleted, or replaced, EndpointSlices are updated to reflect the available backend endpoints.

Clients can continue using the stable Service DNS name and ClusterIP without needing to track individual Pod addresses.

However, Service availability still depends on healthy backend workloads and functioning cluster networking.

## 17. Useful Administration Commands

Inspect Services:

```bash
kubectl get services -n lab-apps
```

Describe Service configuration:

```bash
kubectl describe svc nginx-service -n lab-apps
```

Inspect EndpointSlices:

```bash
kubectl get endpointslices -n lab-apps
```

Inspect CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Check Pod DNS configuration:

```bash
kubectl exec -n lab-apps dns-client -- \
  cat /etc/resolv.conf
```

Test Service DNS:

```bash
kubectl exec -n lab-apps dns-client -- \
  nslookup nginx-service.lab-apps.svc.cluster.local
```

Test internal HTTP:

```bash
kubectl exec -n lab-apps dns-client -- \
  wget -S -O /dev/null http://nginx-service/
```

Inspect application logs:

```bash
kubectl logs -n lab-apps \
  -l app=nginx \
  --prefix=true \
  --max-log-requests=10
```

## 18. Lessons Learned

1. Kubernetes Services provide stable access to changing Pod endpoints.
2. ClusterIP Services are intended primarily for internal cluster communication.
3. Service selectors associate Services with matching Pods.
4. EndpointSlices expose backend endpoint information and readiness conditions.
5. CoreDNS supports internal service discovery.
6. DNS search domains allow short service names within a namespace.
7. Different DNS utilities may behave differently when resolving short names.
8. `kubectl port-forward` is useful for debugging but does not demonstrate Service load balancing.
9. Multiple HTTP connections may be routed to different backend Pods.
10. A small sample of requests does not guarantee an even traffic distribution.
11. Application logs provide useful evidence for tracing network traffic.

## 19. Laboratory Result

**Status: Completed**

The ClusterIP Service was successfully created and validated.

All five Nginx endpoints were ready.

The diagnostic BusyBox Pod resolved the Service DNS name and successfully connected over HTTP.

The traffic distribution experiment confirmed that requests reached all five Nginx replicas through a single Service.

The laboratories established a foundation for further work with readiness probes, ingress routing, network policies, and service troubleshooting.
