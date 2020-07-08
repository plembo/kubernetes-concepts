# Storage

## Volumes
On-disk files in a Container are ephemeral. In Docker, a volume is simply a directory on disk or in another Container. Lifetimes are not managed.

A Kubernetes volume, on the other hand, has an explicit lifetime - the same as the Pod that encloses it. A Kubernetes volume, on the other hand, has an explicit lifetime - the same as the Pod that encloses it.

### Types of Volumes
Part of a much longer list:
* awsElasticBlockStore
* azureDisk
* azureFile
* cephfs
* fc (fibre channel)
* glusterfs
* iscsi
* local
* nfs
* persistentVolumeClaim
* vsphereVolume

## Persistent Volumes

Two API resources:
* PersistentVolume (PV), a piece of storage
* PersistentVolumeClaim (PVC), a request for storage by a user

PV:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

Provisioned statically or dynamically.

Dynamic based on storage class requested in the PVC.

A PVC to PV binding is an exclusive, one-to-one mapping.

Claims will remain unbound indefinitely if a matching volume does not exist. Claims will be bound as matching volumes become available.

Storage Object in Use Protection ensures that PVCs in active use and bound are not removed, to avoid data loss.

You can see that a PVC is protected when the PVC's status is ```Terminating``` and the Finalizers list includes ```kubernetes.io/pvc-protection```:

```bash
$ kubectl describe pvc hostpath
Name:          hostpath
Namespace:     default
StorageClass:  example-hostpath
Status:        Terminating
Volume:
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
               volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
Finalizers:    [kubernetes.io/pvc-protection]
...
```

You can see that a PV is protected when the PV's status is ```Terminating``` and the Finalizers list includes ```kubernetes.io/pv-protection```:

```bash
$ kubectl describe pv task-pv-volume
Name:            task-pv-volume
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Terminating
Claim:
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        1Gi
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/data
    HostPathType:
Events:            <none>
```
When a user is done with a volume, the PVC can be deleted and reclaimed. A Retain policy allows for manual reclaimation.

For volume plugins that support Delete, deletion removes the object from Kubernetes and the underlying platform infrastructure (AWS EBS, GCE PD, Azure Disk).

Recycle policy has been deprecated.

Expanding of PVCs is enabled by default, but the storage class's ```allowVolumeExpansion``` must be set to true. You can only expand a volume if its storage class supports it. Short list of these:
* gcePersistentDisk
* awsElasticBlockStore
* Azure File
* Azure DSisk
* A few others like glusterfs

## Volume Snapshots
In Kubernetes, a VolumeSnapshot represents a snapshot of a volume on a storage system.

A VolumeSnapshotContent is a snapshot taken from a volume in the cluster that has been provisioned by an administrator.

A VolumeSnapshot is a request for snapshot of a volume by a user.

VolumeSnapshotClass allows you to specify different attributes belonging to a VolumeSnapshot.

Snapshot:

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-test
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: pvc-test
```
Snapshot contents:

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotContent
metadata:
  name: snapcontent-72d9a349-aacd-42d2-a240-d775650d2455
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    volumeHandle: ee0cfb94-f8d4-11e9-b2d8-0242ac110002
  volumeSnapshotClassName: csi-hostpath-snapclass
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
    uid: 72d9a349-aacd-42d2-a240-d775650d2455
```
VolumeSnapshotContent object definition:

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotContent
metadata:
  name: new-snapshot-content-test
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    snapshotHandle: 7bdd0de3-aaeb-11e8-9aae-0242ac110002
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
```

## CSI Volume Cloning
The CSI Volume Cloning feature adds support for specifying existing PVCs in the dataSource field to indicate a user would like to clone a Volume.

Clone provisioning:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: clone-of-pvc-1
    namespace: myns
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cloning
  resources:
    requests:
      storage: 5Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: pvc-1
```

## Storage Classes
A StorageClass provides a way for administrators to describe the "classes" of storage they offer. This concept is sometimes called "profiles" in other storage systems.

Storage class resource:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```
Each StorageClass has a provisioner that determines what volume plugin is used for provisioning PVs. This field must be specified.


## Volume Snapshot Classes
VolumeSnapshotClass provides a way to describe the "classes" of storage when provisioning a volume snapshot.

Volume snapshot class object:

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
parameters:
```

Default class:

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
parameters:
```

## Dynamic Volume Provisioning
Dynamic volume provisioning allows storage volumes to be created on-demand. Without dynamic provisioning, cluster administrators have to manually make calls to their cloud or storage provider to create new storage volumes, and then create PersistentVolume objects to represent them in Kubernetes.

Enabling a "slow" class that provisions standard disk-like persistent storage:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```
Enabling a "fast" class to provision SSD-like persistent storage:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

Use dynamic provisioning:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 30Gi
```

Defaulting behavior:

* Mark one StorageClass object as default
* Male sure the DefaultStorageClass admission controller is enabled on the API server

In Multi-Zone clusters, Pods can be spread across Zones in a Region. Single-Zone storage backends should be provisioned in the Zones where Pods are scheduled. This can be accomplished by setting the Volume Binding Mode.

## Node Specific Volume Units

Cloud providers like Google, Amazon, and Microsoft typically have a limit on how many volumes can be attached to a Node. It is important for Kubernetes to respect those limits. Otherwise, Pods scheduled on a Node could get stuck waiting for volumes to attach.

|Cloud Service                  | Max vols per node|
|-------------------------------|------------------|
|Amazon Elastic Block Store     | 39               |
|Google Persistent Disk         | 16               |
|Microsoft Azure Disk Storage   | 16               |

Customize limits by setting the KUBE_MAX_PD_VOLS environment variable and starting the scheduler.

Be careful if setting a limit higher thann the default (check cloud provider's doc).

This limit applies to the cluster, so changing it will affect all nodes.

Dynamic volume limits are supported by:

* Amazon EBS
* Google Persistent Disk
* Azure Disk
* CSI

Kubernetes atomatically determines the node type enforces the appropriate max number of volumes.
Examples:
* Google Compute Engine, up to 127 vols can be atached to a node
* Amazon EBS M5, C5, R5, T3 and Z1D allows only 25 vols. For other EC2, allows 39 vols.
