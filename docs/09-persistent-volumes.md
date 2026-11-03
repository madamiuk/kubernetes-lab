# 09 — Kubernetes PersistentVolumes and PersistentVolumeClaims

## 1. Objective

Learn how Kubernetes manages persistent application data using PersistentVolumes (PV), PersistentVolumeClaims (PVC), and StorageClasses.

The laboratory demonstrates:

- Inspecting available storage provisioners.
- Creating a PersistentVolumeClaim.
- Understanding `WaitForFirstConsumer`.
- Dynamically provisioning a PersistentVolume.
- Mounting persistent storage in a Pod.
- Writing and verifying application data.
- Preserving data after deleting and recreating a Pod.
- Inspecting the physical storage path on a kind node.
- Testing the `Delete` reclaim policy.

**Laboratory:** K8S-019

## 2. Laboratory Environment

| Component | Value |
|---|---|
| Kubernetes context | kind-k8s-lab |
| Namespace | lab-apps |
| StorageClass | standard |
| Provisioner | rancher.io/local-path |
| Reclaim Policy | Delete |
| Volume Binding Mode | WaitForFirstConsumer |
| PVC | lab-data-pvc |
| Requested storage | 1Gi |
| Access mode | ReadWriteOnce |
| Test Pod | storage-test |
| Mount point | /data |
| Node | k8s-lab-control-plane |

The cluster runs locally using kind.

Storage is provided by the `local-path-provisioner`, which creates directories on Kubernetes nodes.

## 3. Initial Storage Inventory

Commands:

```bash
kubectl config current-context
kubectl get storageclass -o wide
kubectl get pv
kubectl get pvc -A
kubectl get csidrivers
kubectl get pods -A | grep -Ei 'storage|provisioner|csi'
```

Observed StorageClass:

```text
NAME                 PROVISIONER           RECLAIMPOLICY   VOLUMEBINDINGMODE
standard (default)   rancher.io/local-path Delete          WaitForFirstConsumer
```

The cluster initially contained no PVs or PVCs.

No CSI drivers were registered.

The `local-path-provisioner` Pod was running in the `local-path-storage` namespace.

## 4. Storage Architecture

The laboratory uses the following storage relationship:

```text
Pod: storage-test
       |
       v
Mount: /data
       |
       v
PVC: lab-data-pvc
       |
       v
PV: pvc-27f2341a-7bd6-47f1-8d6e-7d0bd6c1fff9
       |
       v
StorageClass: standard
       |
       v
Local path on k8s-lab-control-plane
```

A PVC requests storage resources.

A PV represents storage made available to Kubernetes.

A StorageClass describes how storage is provisioned.

## 5. Creating the PersistentVolumeClaim

Manifest:

`manifests/lab-data-pvc.yaml`

Configuration:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-data-pvc
  namespace: lab-apps
  labels:
    app: storage-lab
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```

Validation:

```bash
kubectl apply --dry-run=server \
  -f manifests/lab-data-pvc.yaml
```

Result:

```text
persistentvolumeclaim/lab-data-pvc created (server dry run)
```

## 6. WaitForFirstConsumer

The PVC was created:

```bash
kubectl apply -f manifests/lab-data-pvc.yaml
```

Initial state:

```text
lab-data-pvc   Pending   standard
```

No PersistentVolume existed.

PVC events included:

```text
WaitForFirstConsumer
waiting for first consumer to be created before binding
```

This was expected because the StorageClass uses:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

Kubernetes waits until a Pod requires the volume before selecting a node and provisioning storage.

## 7. Creating the Storage Test Pod

Manifest:

`manifests/storage-test.yaml`

The Pod uses BusyBox and mounts the PVC at `/data`.

Relevant configuration:

```yaml
volumeMounts:
  - name: persistent-data
    mountPath: /data

volumes:
  - name: persistent-data
    persistentVolumeClaim:
      claimName: lab-data-pvc
```

Manifest validation:

```bash
kubectl apply --dry-run=server \
  -f manifests/storage-test.yaml
```

Result:

```text
pod/storage-test created (server dry run)
```

## 8. Dynamic PersistentVolume Provisioning

The Pod was created:

```bash
kubectl apply -f manifests/storage-test.yaml
```

The Pod became Ready:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/storage-test \
  -n lab-apps \
  --timeout=120s
```

The PVC transitioned:

```text
Pending → Bound
```

Provisioned PV:

```text
pvc-27f2341a-7bd6-47f1-8d6e-7d0bd6c1fff9
```

Capacity:

```text
1Gi
```

Access mode:

```text
ReadWriteOnce
```

Kubernetes events included:

```text
ExternalProvisioning
Provisioning
ProvisioningSucceeded
```

The PV was dynamically provisioned by `rancher.io/local-path`.

## 9. Persistent Volume Mount Verification

The Pod ran on:

```text
k8s-lab-control-plane
```

Its initial IP address was:

```text
10.244.0.43
```

Mounted directory verification:

```bash
kubectl exec -n lab-apps storage-test -- \
  ls -ld /data
```

The directory existed and was writable.

Filesystem inspection:

```bash
kubectl exec -n lab-apps storage-test -- \
  df -h /data
```

Observed:

```text
Filesystem    Size    Used    Available    Mounted on
/dev/vdb1      29.4G   2.7G    25.2G        /data
```

Although the PVC requested 1Gi, `df` reported the underlying node filesystem capacity.

The local-path provisioner does not necessarily enforce the requested PVC size as a filesystem quota.

## 10. Writing Test Data

A test file was created in the persistent volume:

```bash
kubectl exec -n lab-apps storage-test -- \
  /bin/sh -c 'printf "%s\n" "K8S-019 PersistentVolume data persistence test" > /data/persistence-test.txt'
```

File verification:

```bash
kubectl exec -n lab-apps storage-test -- \
  ls -l /data/persistence-test.txt

kubectl exec -n lab-apps storage-test -- \
  cat /data/persistence-test.txt
```

Observed file content:

```text
K8S-019 PersistentVolume data persistence test
```

File size:

```text
47 bytes
```

## 11. SHA-256 Integrity Verification

A SHA-256 checksum was calculated:

```bash
kubectl exec -n lab-apps storage-test -- \
  sha256sum /data/persistence-test.txt
```

Reference checksum:

```text
cab2a0ef4a1421df981981245f7a9267756b6a7505e207cc2517a1fda16b3dca
```

The checksum was saved for comparison after deleting and recreating the Pod.

## 12. Deleting the Pod Without Deleting Storage

Original Pod UID:

```text
d8ce5d16-929e-47f0-82b1-a111dd8bf957
```

The Pod was deleted:

```bash
kubectl delete pod storage-test \
  -n lab-apps \
  --wait=true \
  --timeout=120s
```

The Pod disappeared, but the PVC and PV remained `Bound`.

PVC inspection showed:

```text
Used By: <none>
Unused: True
Reason: NoPodsUsingPVC
```

The volume was no longer mounted by an active Pod, but the storage resource remained available.

## 13. Recreating the Pod

The original Pod manifest was applied again:

```bash
kubectl apply -f manifests/storage-test.yaml
```

The recreated Pod became Ready.

New Pod UID:

```text
834556d3-810e-4e3d-8664-08a16701736a
```

New Pod IP:

```text
10.244.0.44
```

This confirmed that Kubernetes created a new Pod object.

The PVC and PV remained unchanged.

## 14. Verifying Data Persistence

The recreated Pod successfully accessed:

```text
/data/persistence-test.txt
```

The file size remained 47 bytes.

The file content was unchanged.

The SHA-256 checksum was recalculated:

```bash
kubectl exec -n lab-apps storage-test -- \
  sha256sum /data/persistence-test.txt
```

Result:

```text
cab2a0ef4a1421df981981245f7a9267756b6a7505e207cc2517a1fda16b3dca
```

The checksum matched the original value.

**Result:** Data persisted across Pod deletion and recreation.

## 15. Inspecting the PersistentVolume

StorageClass inspection:

```bash
kubectl describe storageclass standard
```

PersistentVolume inspection:

```bash
PV=$(kubectl get pvc lab-data-pvc \
  -n lab-apps \
  -o jsonpath='{.spec.volumeName}')

kubectl get pv "$PV" \
  -o jsonpath='ReclaimPolicy={.spec.persistentVolumeReclaimPolicy}{"\n"}StorageClass={.spec.storageClassName}{"\n"}VolumeType={.spec.hostPath.path}{"\n"}NodeAffinity={.spec.nodeAffinity}{"\n"}'
```

Observed:

```text
ReclaimPolicy=Delete
StorageClass=standard
```

The PV used a `hostPath` volume and node affinity restricted it to:

```text
k8s-lab-control-plane
```

## 16. Inspecting Storage on the Kind Node

The physical path was:

```text
/var/local-path-provisioner/pvc-27f2341a-7bd6-47f1-8d6e-7d0bd6c1fff9_lab-apps_lab-data-pvc
```

The path was retrieved from the PV:

```bash
PV_PATH=$(kubectl get pv "$PV" \
  -o jsonpath='{.spec.hostPath.path}')
```

Directory inspection:

```bash
docker exec k8s-lab-control-plane \
  ls -la "$PV_PATH"
```

The directory contained:

```text
persistence-test.txt
```

Checksum inspection on the node:

```bash
docker exec k8s-lab-control-plane \
  sha256sum "$PV_PATH/persistence-test.txt"
```

The checksum matched the value calculated inside the Pod.

This verified that the Pod and node accessed the same stored data.

## 17. Understanding ReclaimPolicy

The StorageClass and PV both used:

```text
ReclaimPolicy: Delete
```

This policy applies when a PersistentVolume is released from its claim.

Deleting a Pod does not itself release the PVC or trigger deletion of the PV.

Deleting the PVC allows Kubernetes and the external provisioner to reclaim the PV and associated storage.

By comparison, `Retain` preserves the underlying storage resource for administrator-managed recovery or reuse.

`Retain` does not provide a backup or guarantee data recovery after storage failure.

## 18. Testing ReclaimPolicy Delete

Before deleting the PVC, the diagnostic Pod was removed:

```bash
kubectl delete pod storage-test \
  -n lab-apps \
  --wait=true \
  --timeout=120s
```

The PVC and PV remained Bound.

The file was still present on the node.

The PVC was then deleted:

```bash
kubectl delete pvc lab-data-pvc \
  -n lab-apps \
  --wait=true \
  --timeout=120s
```

Verification:

```bash
kubectl get pvc lab-data-pvc \
  -n lab-apps \
  --ignore-not-found

kubectl get pv "$PV" --ignore-not-found
```

Both commands returned no resources.

The directory was checked using:

```bash
docker exec k8s-lab-control-plane \
  /bin/sh -c '
    if [ -d "$1" ]; then
      echo "DIRECTORY STILL EXISTS"
    else
      echo "DIRECTORY REMOVED"
    fi
  ' sh "$PV_PATH"
```

Observed result:

```text
DIRECTORY REMOVED
```

**Result:** The `local-path-provisioner` removed the PV and its backing directory after PVC deletion.

## 19. Operational Lessons Learned

1. A PVC provides a stable reference to storage independently of a Pod.
2. `WaitForFirstConsumer` delays storage provisioning until a consuming Pod is scheduled.
3. A dynamic provisioner creates PVs when required.
4. `ReadWriteOnce` limits mounting to a single node, not necessarily a single Pod.
5. PVC requested capacity does not always equal an enforced filesystem quota.
6. Data stored on a PV can survive deletion and recreation of a Pod.
7. SHA-256 checksums provide a practical method of verifying data integrity.
8. `hostPath` and local-path storage are tied to specific Kubernetes nodes.
9. `NodeAffinity` restricts which nodes can use the volume.
10. Deleting a Pod does not automatically delete its PVC or PV.
11. The `Delete` reclaim policy can remove the backing storage after PVC deletion.
12. Kubernetes persistence does not replace backups.
13. Local storage is not automatically resilient to node failure.
14. Destructive storage operations should be performed only after verifying resource ownership and dependencies.

## 20. Laboratory Result

**Status: Technical experiments completed**

Successfully configured a 1Gi PersistentVolumeClaim using the default `standard` StorageClass and `rancher.io/local-path` provisioner.

Observed the PVC transition from `Pending` to `Bound` after creating the first consuming Pod.

Verified the PersistentVolume mount and wrote a test file.

Deleted and recreated the Pod while preserving the PVC and PV.

Confirmed data integrity using matching SHA-256 checksums before and after recreation.

Inspected the backing directory directly on the kind node.

Confirmed that deleting the PVC with `ReclaimPolicy: Delete` removed both the PV and its backing directory.

The main Nginx Deployment remained available with five ready replicas throughout the laboratory.

Documentation and Git publication remain the final project housekeeping steps.
