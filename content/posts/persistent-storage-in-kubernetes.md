---
title: "Persistent Storage in Kubernetes: Volumes, PersistentVolumes, Claims, StorageClasses and CSI"
date: 2026-11-17
draft: true
description: "How storage works in Kubernetes: volume types, PersistentVolumes and PersistentVolumeClaims, access modes, reclaim policies, StorageClasses, dynamic provisioning, StatefulSets, CSI drivers, ephemeral volumes, snapshots, volume expansion, keeping data safe and debugging a claim stuck in Pending."
summary: "Containers forget everything when they restart, and Pods move between nodes. Databases don't like either. Here is how Kubernetes gives workloads storage that outlives them, the rules that decide whether a claim binds, and what happens between a claim and a mounted disk."
tags: ["kubernetes", "storage", "persistent volumes", "statefulset", "csi"]
categories: ["Programming"]
---

A container's filesystem lives only as long as the container. When it crashes and restarts, everything it wrote is gone. A Pod is not much more durable: it can be deleted, evicted or rescheduled to another node at any moment. For a web server that's a feature. For a database, a message queue or anything else that keeps state, it's a problem. The data has to outlive the container, the Pod and sometimes the node.

Kubernetes solves this with a small set of objects that separate *asking for storage* from *providing it*. Once the binding rules click, it's predictable. Until then, you can spend a long time staring at a claim that will never bind. This post covers the whole picture: from volumes inside a Pod, through PersistentVolumes, claims and StorageClasses, to StatefulSets, the CSI drivers that do the real work, and how to keep your data safe.

The examples match Kubernetes v1.37.

## The mental model

There are four objects, and two people who care about them.

```text
   developer's side                      administrator's side
 ┌──────────────────┐               ┌────────────────────────────────┐
 │ Pod              │               │ StorageClass                   │
 │  volumes:        │               │  provisioner, parameters,      │
 │   - pvc: data    │               │  reclaimPolicy, bindingMode    │
 └────────┬─────────┘               └───────────────┬────────────────┘
          │ uses                                    │ creates (dynamic)
 ┌────────▼─────────┐    binds 1:1   ┌──────────────▼─────────────────┐
 │ PersistentVolume │◄──────────────►│ PersistentVolume               │
 │ Claim            │               │ (cluster-scoped)               │
 │  (namespaced)    │               │  capacity, modes, class,       │
 │  size,modes,class│               │  reclaim policy, backend       │
 └──────────────────┘               └───────────────┬────────────────┘
                                                    │ points at
                                         ┌──────────▼──────────┐
                                         │ real storage: NFS,  │
                                         │ disk, CSI volume... │
                                         └─────────────────────┘
```

- A **PersistentVolume (PV)** is a piece of storage that exists in the cluster, created by an administrator or by a provisioner. It's cluster-scoped, like a Node, and its lifecycle is independent of any Pod.[^1]
- A **PersistentVolumeClaim (PVC)** is a request for storage: "I want 1Gi, read-write, from this class." It's namespaced. Pods consume node resources; PVCs consume PV resources.[^1]
- A **StorageClass** describes a kind of storage the cluster offers and, through its provisioner, creates PVs on demand.[^2]
- A **Pod** never references a PV directly. It references a claim.

Everything else in this domain is detail on how these connect.

## Volumes in a Pod

Before PersistentVolumes, the basics. A Pod declares `volumes` at the Pod level, and each container mounts the ones it needs with `volumeMounts`. The `name` is what links them.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do date >> /data/log; sleep 5; done"]
    volumeMounts:
    - name: scratch
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "tail -f /data/log"]
    volumeMounts:
    - name: scratch
      mountPath: /data
      readOnly: true
  volumes:
  - name: scratch
    emptyDir: {}
```

The volume types you should know without looking them up:

**`emptyDir`.** Created empty when the Pod is scheduled to a node, deleted when the Pod is removed. It survives container restarts, not Pod deletion. Use it for scratch space and for sharing files between containers in one Pod, as above. Set `medium: Memory` to back it with tmpfs, and `sizeLimit` to cap it.[^3]

```yaml
  volumes:
  - name: cache
    emptyDir:
      medium: Memory
      sizeLimit: 500Mi
```

**`hostPath`.** Mounts a file or directory from the node's filesystem. The data stays on that node, so a Pod rescheduled elsewhere won't see it. It needs no storage backend, which makes it handy on a single-node test cluster, but it's a security risk in real clusters: a Pod with a hostPath volume can read, and often write, the node's own files. The optional `type` field checks or creates the path: `DirectoryOrCreate`, `Directory`, `FileOrCreate`, `File`, `Socket`, `CharDevice` or `BlockDevice`.[^3]

```yaml
  volumes:
  - name: logs
    hostPath:
      path: /var/log/app
      type: DirectoryOrCreate
```

**`configMap`, `secret`, `downwardAPI` and `projected`.** These expose Kubernetes data as files. A ConfigMap volume is always mounted read-only, and a container that mounts it with `subPath` won't see later updates to the ConfigMap.[^3] A `projected` volume combines several of these sources into one directory.[^4]

**`persistentVolumeClaim`.** Mounts the PV bound to a claim. This is the one the rest of this post is about.

## PersistentVolumes

Here is the simplest possible PV, a static one backed by a hostPath. It's fine on a test cluster and a good way to see the fields:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

There's no `kubectl create pv` command, so you write this YAML yourself. The "Configure a Pod to Use a PersistentVolume for Storage" task page in the docs[^5] has a good example to start from.

The fields that matter:

| Field | What it does | Default |
|---|---|---|
| `capacity.storage` | Size of the volume. Storage is currently the only resource a PV has. | none, required |
| `accessModes` | How the volume can be mounted (next section). | none, required |
| `persistentVolumeReclaimPolicy` | What happens when the claim is deleted. | `Retain` for manual PVs; dynamic ones inherit from the class |
| `storageClassName` | The class this PV belongs to. Only claims for the same class can bind. | no class |
| `volumeMode` | `Filesystem` (mounted as a directory) or `Block` (raw device). | `Filesystem` |
| `nodeAffinity` | Restricts which nodes can use the volume. Required for `local` volumes. | none |
| `mountOptions` | Extra mount flags, for `nfs`, `iscsi` and `csi`. Not validated. | none |

### Access modes

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by a **single node**. Several Pods on that node can still share it. |
| `ReadOnlyMany` | ROX | Read-only by many nodes. |
| `ReadWriteMany` | RWX | Read-write by many nodes. |
| `ReadWriteOncePod` | RWOP | Read-write by a **single Pod** in the whole cluster. CSI volumes only. |

Two details worth remembering, both straight from the docs:[^1]

1. **RWO is per node, not per Pod.** Two Pods on the same node can both write to an RWO volume. If only one Pod may ever use the volume, use `ReadWriteOncePod`.
2. **Access modes are for matching, not for enforcement.** Kubernetes uses them to pair claims with volumes. Apart from RWOP, they don't write-protect anything after mounting: a PV labelled ROX isn't guaranteed to be read-only. To actually mount read-only, set `readOnly: true` on the `volumeMount` or the `persistentVolumeClaim` volume source.

A volume can only be mounted with one access mode at a time, even if it supports several. And which modes are possible depends on the backend: `hostPath` supports only RWO, NFS supports RWO, ROX and RWX, and for CSI it depends on the driver.[^1]

### Reclaim policies

When a claim is deleted, its PV is released, and the reclaim policy decides what happens next:[^1]

- **`Retain`**: the PV stays, in phase `Released`, with the old data on it. It is **not** available to a new claim until an administrator deals with it.
- **`Delete`**: the PV object and the backing storage (the cloud disk, the CSI volume) are deleted. This is the default for dynamically provisioned volumes, because they inherit their StorageClass's policy, and the StorageClass default is `Delete`.
- **`Recycle`**: deprecated. It ran `rm -rf` on the volume and made it available again. In 1.37 only `nfs` and `hostPath` support it. You may still see it in old examples; the recommendation is dynamic provisioning instead.

Changing the policy of an existing PV, for example to protect a dynamically provisioned volume's data before deleting its claim, is a one-line patch:[^6]

```bash
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### Phases

A PV is always in one of four phases:[^1]

- **`Available`**: free, not bound to a claim
- **`Bound`**: bound to a claim
- **`Released`**: the claim was deleted, but the resource hasn't been reclaimed
- **`Failed`**: automatic reclamation failed

The one that trips people up is `Released`. A `Retain` PV in `Released` still has a `claimRef` pointing at the deleted claim, so nothing else can bind to it. To reuse it, clear the reference:

```bash
kubectl patch pv pv-log --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'
```

The PV goes back to `Available` and any matching claim can bind. The old data is still on it, so only do this if that's what you want.

## PersistentVolumeClaims

A claim that binds to the PV above:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

### How binding works

A control loop watches for new claims, finds a matching PV and binds them. The binding is one-to-one and exclusive, recorded in the PV's `claimRef`.[^1] For a static PV to match a claim, **all** of these must line up:

1. **Storage class.** Same `storageClassName` on both, or no class on both.
2. **Access modes.** The PV must offer the modes the claim asks for.
3. **Size.** The PV's capacity must be at least the claim's request.
4. **Volume mode.** `Filesystem` with `Filesystem`, `Block` with `Block`. An unspecified mode counts as `Filesystem`.
5. **Selector.** If the claim has a label `selector`, the PV's labels must match it.
6. **Availability.** The PV must be `Available`, not bound or released.

Note rule 3: the claim gets **at least** what it asked for. The 500Mi claim above binds to the 1Gi PV, and `kubectl get pvc` then reports a capacity of 1Gi, because the whole PV is now taken. If no PV matches, the claim stays `Pending` indefinitely, and binds as soon as a matching PV appears.

### The `storageClassName` trap

This is the single most common reason a claim doesn't bind. These three are different:[^1]

- **`storageClassName: manual`**: binds only to PVs of class `manual`, or is provisioned by the `manual` StorageClass.
- **`storageClassName: ""`**: explicitly no class. Binds only to PVs with no class, and never triggers dynamic provisioning.
- **No `storageClassName` field at all**: if the cluster has a default StorageClass, the admission controller fills in that default. The claim then only binds to PVs of the default class, or gets a new volume provisioned.

So if you write a PV with no class, and a claim with no class, in a cluster that has a default StorageClass, they will **not** bind: the claim silently becomes a claim for the default class. Either give both the same named class, or set `storageClassName: ""` explicitly on the claim.

There's also a retroactive version of this. A claim created with no class while the cluster has no default stays unset, and when a default StorageClass appears, Kubernetes updates the claim to use it. Claims with `""` are never updated.[^1]

When you want a claim to bind to an existing PV, check the PV's class first (`kubectl get pv` shows it in the `STORAGECLASS` column) and copy it.

### Binding to a specific PV

To make a claim bind to one particular PV, set `volumeName`:[^1]

```yaml
spec:
  storageClassName: ""   # or the PV's class; must still match
  volumeName: pv-log
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
```

This skips some matching (node affinity, for instance), but the class, access modes and size still have to be valid. If the named PV is already bound to another claim, this claim stays `Pending`. To stop other claims grabbing the PV first, reserve it from the other side by setting `claimRef` on the PV:

```yaml
spec:
  claimRef:
    name: claim-log
    namespace: default
```

### Using the claim in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: logger
  namespace: default
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
  volumes:
  - name: log-volume
    persistentVolumeClaim:
      claimName: claim-log
```

The claim must be in the **same namespace** as the Pod.[^1] PVs are cluster-scoped, but a claim is not, so you can't mount a claim from another namespace. For a Deployment, the same `volumes` block goes in the Pod template, and every replica mounts the same claim. That's usually not what a stateful application wants, which is where StatefulSets come in.

### StatefulSets: one volume per replica

Scale that Deployment to three replicas and all three Pods reference the same claim. With an RWO volume, Pods on other nodes can't mount it and get stuck in `ContainerCreating`. Pods on the same node *can* mount it, and then three database processes write to the same files, which is worse. Even with RWX storage, most databases expect each instance to have its own data directory.

A **StatefulSet** fixes this with `volumeClaimTemplates`. Instead of naming one claim, you describe a claim, and the controller creates one per replica:[^13]

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:17
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: fast
      resources:
        requests:
          storage: 20Gi
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
```

This creates the claims `data-db-0`, `data-db-1` and `data-db-2`, named `<template>-<statefulset>-<ordinal>`. The binding is sticky. If `db-1` is deleted or its node fails, the replacement Pod is also called `db-1` and mounts `data-db-1` again, wherever it's scheduled (for a zonal disk, the scheduler keeps it in the right zone).

By default, the claims outlive the StatefulSet. Deleting the StatefulSet or scaling it down leaves them, and their data, in place, so scaling back up reattaches the same volumes. The `persistentVolumeClaimRetentionPolicy` makes this explicit:[^13]

- **`whenDeleted`**: what happens to the claims when the StatefulSet is deleted.
- **`whenScaled`**: what happens to the claims of replicas removed by scaling down.

Both accept `Retain` (the default) or `Delete`. `Delete` removes the claims, and with a `Delete` reclaim policy that takes the backing volumes and the data with them. Only choose it for data you can rebuild.

You can't edit `volumeClaimTemplates` on an existing StatefulSet. To grow the volumes, expand each claim directly (see "Expanding a claim" below).

### Protection

You can't pull storage out from under a running Pod. A claim deleted while a Pod uses it stays in `Terminating`, with the `kubernetes.io/pvc-protection` finalizer, until the Pod is gone. A PV deleted while bound stays `Terminating` with `kubernetes.io/pv-protection` until its claim is gone.[^1] If `kubectl delete pvc` seems to hang, that's why: delete the Pod (or scale down the Deployment) first.

New in 1.37, as a beta feature enabled by default: claims get an `Unused` condition that records whether any non-terminal Pod references them, and since when. It's a handy way to find claims nothing has used in months.[^1]

## StorageClasses and dynamic provisioning

Creating PVs by hand doesn't scale. With dynamic provisioning, the administrator creates a StorageClass once, and a claim that asks for that class gets a new PV created for it automatically.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

The fields:[^2]

- **`provisioner`**: which volume plugin creates the volumes. Usually a CSI driver's name. The in-tree cloud provisioners are gone (the AWS EBS one was removed in v1.27), so expect CSI driver names.
- **`parameters`**: passed to the provisioner. They're driver-specific, which means you don't need to memorise any.
- **`reclaimPolicy`**: copied to every PV this class creates. `Delete` if unset.
- **`allowVolumeExpansion`**: whether claims of this class can be resized.
- **`volumeBindingMode`**: when to bind and provision (below).
- **`mountOptions`**: copied to the PVs.

A StorageClass is mostly immutable: to change the provisioner, parameters or reclaim policy, delete it and create it again. Existing PVs aren't affected.

### `Immediate` vs `WaitForFirstConsumer`

With the default, **`Immediate`**, a volume is provisioned and bound the moment the claim is created. That's a problem for storage that only exists in some places: a zonal disk gets created in a zone, and then the Pod that needs it can't be scheduled there.

**`WaitForFirstConsumer`** delays binding and provisioning until a Pod using the claim is scheduled. The scheduler picks a node considering the Pod's resource requests, node selectors, affinity and taints, and the volume is created to match.[^2] Two practical consequences:

- A claim of a `WaitForFirstConsumer` class sits in `Pending` until a Pod uses it, and `kubectl describe pvc` shows an event like `waiting for first consumer to be created before binding`. **That's normal, not a fault.** Don't try to "fix" it; create the Pod.
- Don't use `nodeName` in a Pod with such a claim. `nodeName` bypasses the scheduler, so the claim never binds and the Pod is stuck. Use a `nodeSelector` on `kubernetes.io/hostname` instead.[^2]

### Local volumes

`local` volumes are disks attached to specific nodes. There's no dynamic provisioner for them in Kubernetes itself, so you pair static PVs with a class that doesn't provision anything:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-node01
spec:
  capacity:
    storage: 10Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values: [node01]
```

A `local` PV **must** have `nodeAffinity`, so the scheduler knows which node the data lives on.[^1][^3] `WaitForFirstConsumer` lets the scheduler choose a PV on a node where the Pod can actually run. Unlike `hostPath`, the scheduler understands where a `local` volume is, which is why the docs recommend it over `hostPath` for anything beyond a single-node test.

### The default StorageClass

A claim with no `storageClassName` gets the default class. The default is whichever class carries the annotation `storageclass.kubernetes.io/is-default-class: "true"`. To change it, remove the annotation from the old default and add it to the new one:[^7]

```bash
kubectl get storageclass          # the default shows "(default)" after its name

kubectl patch storageclass standard \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl patch storageclass fast \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Aim for exactly one default. If there are several, the most recently created one wins.[^2] A cluster can also have none, in which case claims without a class only bind to PVs with no class.

## Volume drivers and CSI

Every PV and every StorageClass ultimately names a **driver**: the code that actually creates, attaches and mounts storage. You'll rarely write one, but when storage breaks, knowing how a driver fits together tells you where to look.

### In-tree, FlexVolume and CSI

Kubernetes has had three generations of volume plugins:

- **In-tree plugins** were compiled into Kubernetes itself: `awsElasticBlockStore`, `gcePersistentDisk`, `azureDisk`, `cinder` and so on. Any bug fix or new storage system had to wait for a Kubernetes release. The cloud ones are now deprecated and transparently redirected to their CSI drivers ("CSI migration"); some, like `cephfs`, `rbd` and `glusterfs`, are gone entirely.[^1] The in-tree types that remain and that you'll still use are the simple ones: `emptyDir`, `hostPath`, `local`, `nfs`, `iscsi`, `fc`, and the ConfigMap/Secret family.
- **FlexVolume** was an early attempt at out-of-tree drivers: an executable installed on every node. It needed root access to the nodes' filesystems to deploy, and it has been deprecated since v1.23.[^1]
- **CSI**, the Container Storage Interface, is the standard today. It's a gRPC specification shared by several orchestrators, and a CSI driver is just a set of containers you deploy on the cluster like any other workload. Storage vendors ship and update drivers on their own schedule.[^8]

So in practice, "the storage driver" means "the CSI driver", and the `provisioner` in a StorageClass is a CSI driver's name, like `ebs.csi.aws.com`, `pd.csi.storage.gke.io` or `nfs.csi.k8s.io`.

### How a CSI driver is deployed

A CSI driver normally runs as two components:[^9]

- **The controller plugin**, a Deployment or StatefulSet running anywhere in the cluster. It contains the vendor's driver container, which implements the CSI *Controller* service (create, delete, attach, expand volumes on the backend), plus a set of standard **sidecar** containers maintained by the Kubernetes CSI project. The sidecars watch the Kubernetes API and translate what they see into gRPC calls to the driver, over a Unix socket shared through an `emptyDir`.
- **The node plugin**, a DaemonSet with one Pod per node. It contains the driver container implementing the CSI *Node* service (format, mount and unmount on that node), plus the `node-driver-registrar` sidecar, which registers the driver with the kubelet. It runs privileged, because it has to mount things on the host, and shares sockets and mount directories with the kubelet through `hostPath` volumes under `/var/lib/kubelet`.

The sidecars are worth knowing by name, because their logs are where storage failures show up:

| Sidecar | Runs in | Watches | Calls on the driver |
|---|---|---|---|
| `external-provisioner` | controller | PVCs | `CreateVolume`, `DeleteVolume` |
| `external-attacher` | controller | VolumeAttachments | `ControllerPublishVolume`, `ControllerUnpublishVolume` |
| `external-resizer` | controller | PVC size changes | `ControllerExpandVolume` |
| `external-snapshotter` (`csi-snapshotter`) | controller | VolumeSnapshotContents | `CreateSnapshot`, `DeleteSnapshot` |
| `node-driver-registrar` | node | (registers with the kubelet) | `GetPluginInfo`, `NodeGetInfo` |
| `livenessprobe` | both | (health endpoint) | `Probe` |

The controller sidecars talk to the API server, so each needs its own RBAC rules; driver Helm charts and manifests include them.[^9] A driver doesn't have to ship every sidecar: no `external-resizer` means no expansion, no snapshotter means no snapshots.

### What happens from claim to mount

Here's the full path of a dynamically provisioned CSI volume, and which piece does each step:

```text
 1. PVC created (class "fast", provisioner ebs.csi.aws.com)
        │  external-provisioner sees it ──► CreateVolume ──► backend creates disk
        ▼
 2. PV created by the provisioner (spec.csi.volumeHandle = disk ID), PVC Bound
        │  (with WaitForFirstConsumer, steps 1–2 wait until a Pod is scheduled)
        ▼
 3. Pod scheduled to node01
        │  attach/detach controller (in kube-controller-manager) creates a
        │  VolumeAttachment ──► external-attacher ──► ControllerPublishVolume
        │  ──► disk attached to node01        (skipped if attachRequired: false)
        ▼
 4. kubelet on node01 ──► node plugin: NodeStageVolume (format + mount once per node)
                                     NodePublishVolume (bind-mount into the Pod)
        ▼
 5. Container starts with the volume at its mountPath
```

Deletion runs the same chain backwards: unpublish and unstage on the node, detach, and, if the reclaim policy is `Delete`, `DeleteVolume` on the backend.

This is the most useful picture for troubleshooting. A claim stuck in `Pending` is a step 1 problem: look at the provisioner. A Pod stuck in `ContainerCreating` with `FailedAttachVolume` is step 3: look at the VolumeAttachment and the attacher. `FailedMount` is step 4: look at the node plugin on that node and the kubelet.

### The CSI API objects

Four cluster-scoped objects in the `storage.k8s.io` group describe the drivers in a cluster:[^10]

- **`CSIDriver`**: one per driver, describing how Kubernetes should treat it. Fields include `attachRequired` (whether step 3 happens at all; NFS-style drivers set it to `false`), `podInfoOnMount` (pass the Pod's name and namespace to the driver at mount time), `fsGroupPolicy`, and `volumeLifecycleModes` (`Persistent`, `Ephemeral` or both). The driver's install manifests create it.
- **`CSINode`**: one per node, with the same name as the node, listing the CSI drivers registered there and their node IDs. The kubelet fills it in when `node-driver-registrar` registers a driver. It can also record how many volumes of each driver the node can attach, which the scheduler respects.
- **`VolumeAttachment`**: one per volume attached to a node, created by the attach/detach controller. Its `status.attached` and `attachError` fields show how step 3 went.
- **`CSIStorageCapacity`**: optional, published by drivers that report free capacity per topology segment, so the scheduler avoids nodes where provisioning would fail.

These commands answer most "what's installed and is it working" questions:

```bash
kubectl get csidrivers                          # which drivers are installed
kubectl get csinodes                            # which nodes have which drivers registered
kubectl describe csinode node01                 # driver list and attach limits for one node
kubectl get volumeattachments                   # what's attached where, and whether it worked
kubectl get storageclass -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner

# Find the driver's Pods (namespace and labels vary by driver), then read the sidecar logs:
kubectl get pods -A | grep -i csi
kubectl -n kube-system logs <controller-pod> -c csi-provisioner
kubectl -n kube-system logs <node-pod-on-node01> -c <driver-container>
```

If a node is missing from a driver's `CSINode` entry, the node plugin isn't running or didn't register there, and every Pod that needs that driver on that node will fail to mount.

### Three ways to use a CSI driver

**1. Through a StorageClass.** The normal way, covered above: the class's `provisioner` is the driver's name, `parameters` are passed to `CreateVolume`, and claims of that class get volumes provisioned on demand.

**2. As a pre-provisioned PV.** If the volume already exists on the backend, for example a disk restored from backup, write the PV yourself with a `csi` source pointing at it:[^3]

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: existing-disk
spec:
  capacity:
    storage: 50Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast
  csi:
    driver: ebs.csi.aws.com            # must match the CSIDriver name
    volumeHandle: vol-0abc123def456     # the backend's ID for the volume
    fsType: ext4
    volumeAttributes: {}                # driver-specific, optional
```

`driver` and `volumeHandle` are required. The optional `*SecretRef` fields (`controllerPublishSecretRef`, `nodeStageSecretRef`, `nodePublishSecretRef`, `controllerExpandSecretRef`, `nodeExpandSecretRef`) point at Secrets with credentials the driver needs for the matching call. Then bind a claim to it with `volumeName`, as described earlier.

**3. Inline, as an ephemeral volume.** Two flavours, both declared in the Pod spec and deleted with the Pod:[^11]

A **generic ephemeral volume** works with any driver that supports dynamic provisioning. You put a claim template inside the Pod, and Kubernetes creates a real PVC named `<pod>-<volume>`, owned by the Pod, so it's garbage-collected when the Pod goes. It's an `emptyDir` with the features of real storage: a fixed size, a StorageClass of your choice, snapshots and resizing if the driver supports them.

```yaml
  volumes:
  - name: scratch
    ephemeral:
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: fast
          resources:
            requests:
              storage: 5Gi
```

Because the PVC is created on the user's behalf, anyone who can create Pods can effectively create PVCs. Namespace quotas on PVCs still apply.[^11]

A **CSI ephemeral volume** skips PVs and claims altogether: the kubelet asks the node plugin to set up a volume directly, using `volumeAttributes` from the Pod spec. Only drivers that list `Ephemeral` in their `CSIDriver`'s `volumeLifecycleModes` support it. It's meant for small, node-local volumes that inject data, like the Secrets Store CSI driver mounting secrets from an external vault.[^11]

```yaml
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: my-vault-secrets
```

### Snapshots

Volume snapshots are a CSI-only feature, and unlike everything else in this post they are **not** in the core API. `VolumeSnapshot`, `VolumeSnapshotContent` and `VolumeSnapshotClass` are CRDs, and installing them plus the snapshot controller is the job of the distribution, or yours on a kubeadm cluster. The driver also needs the `csi-snapshotter` sidecar.[^12]

The objects mirror the PVC/PV/StorageClass trio:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snap
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: claim-data
```

To restore, create a new claim with the snapshot as its `dataSource`. To clone a volume instead, use an existing PVC as the `dataSource`.[^1] If `kubectl get volumesnapshot` returns `the server doesn't have a resource type "volumesnapshot"`, the CRDs aren't installed.

## Expanding a claim

If the claim's StorageClass has `allowVolumeExpansion: true`, you grow a volume by editing the **claim**, not the PV:[^1]

```bash
kubectl patch pvc claim-data -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

The existing volume is resized; no new PV is created. A few rules:

- **Only bigger.** Kubernetes doesn't support shrinking a claim below its current size.
- **Don't edit the PV's capacity by hand.** If you change the PV and then the claim to match, the control plane decides there's nothing to do, and the backing storage is never resized.
- **The filesystem grows when the volume is in use.** For volumes with a filesystem (XFS, ext3, ext4), the resize finishes when a Pod mounts the claim, or online while it's running if the driver supports it. Watch `kubectl describe pvc` for the conditions and events.
- **The driver must support it.** For CSI volumes, expansion needs the driver to implement it and the `external-resizer` sidecar to be deployed. If the claim's size never changes, check the resizer's logs.

Static `hostPath` PVs can't be expanded at all.

## Keeping data safe

Kubernetes makes it easy to create storage, and just as easy to destroy it. A few habits prevent the worst surprises:

- **Know your reclaim policy.** Dynamically provisioned volumes inherit `Delete` from their StorageClass by default.[^2] Delete the claim, or the namespace that contains it, and the backing disk goes with it, with no confirmation. For data that matters, use a StorageClass with `reclaimPolicy: Retain`, or patch the existing PVs to `Retain`, as shown above.[^6] A retained PV survives its claim in phase `Released`, and you can recover the data or bind it to a new claim.
- **Check StatefulSet retention.** A `persistentVolumeClaimRetentionPolicy` of `Delete` removes claims on scale-down or deletion.[^13]
- **Snapshots are not backups.** A CSI snapshot usually lives in the same storage system as the volume, often in the same account and region. It protects you from a bad deploy or an accidental `rm`, not from losing the backend or the cloud account. Copy important data somewhere else.[^12]
- **Make backups consistent.** A snapshot of a running database's disk is like pulling the power cord: the database will usually recover, but not always. For a clean backup, pause writes or flush the database first, or use its own dump tool (`pg_dump`, `mysqldump` and so on).
- **Back up the objects too.** A disk without its PV, claim and the workload that uses it is hard to restore. Tools like Velero back up Kubernetes objects together with volume data, through CSI snapshots or file-level copies to object storage, and restore them into the same or another cluster.[^14]
- **Test the restore.** A backup you have never restored is a hope, not a backup.

## Debugging a claim stuck in `Pending`

Start with the events:

```bash
kubectl describe pvc <name>
kubectl get pv
kubectl get storageclass
```

Then work down the list:

| Symptom in `describe pvc` | Likely cause | Fix |
|---|---|---|
| `waiting for first consumer to be created before binding` | `WaitForFirstConsumer` class, no Pod yet | Create the Pod. This is normal. |
| `no persistent volumes available for this claim and no storage class is set` | No matching PV, and dynamic provisioning disabled | Create a matching PV, or set a class |
| `storageclass.storage.k8s.io "x" not found` | Typo in `storageClassName`, or class missing | Fix the name or create the class |
| `waiting for a volume to be created ... by external provisioner` | Provisioner not running or failing | Check the CSI driver's Pods and logs |
| No events, PV exists but stays `Available` | One of the six matching rules fails | Compare class, modes, size, volumeMode, selector |
| PV is `Released`, claim still `Pending` | Old `claimRef` on a `Retain` PV | Remove the `claimRef` |

For the "PV exists but won't bind" case, put them side by side:

```bash
kubectl get pv,pvc -o custom-columns=\
KIND:.kind,NAME:.metadata.name,CLASS:.spec.storageClassName,\
MODES:.spec.accessModes,SIZE:.spec.capacity.storage,REQ:.spec.resources.requests.storage,\
VMODE:.spec.volumeMode
```

Nine times out of ten the difference is the class (an empty one versus the default) or an access mode.

And when a **Pod** is `Pending` or `ContainerCreating` because of storage, `kubectl describe pod` shows why:

| Event on the Pod | Where it broke | Where to look |
|---|---|---|
| `pod has unbound immediate PersistentVolumeClaims` | The claim isn't bound | The claim, using the table above |
| `persistentvolumeclaim "x" not found` | Wrong claim name, or claim in another namespace | `kubectl get pvc -n <pod namespace>` |
| `FailedAttachVolume` | Attaching the volume to the node | `kubectl get volumeattachments`, the `csi-attacher` logs |
| `FailedMount` / `MountVolume.SetUp failed` | Mounting on the node | The CSI node plugin Pod on that node, `kubectl describe csinode <node>`, the kubelet logs |
| `driver name x not found in the list of registered CSI drivers` | Node plugin not running or not registered | The driver's DaemonSet on that node |

## Working habits

- **Keep the docs close.** You can't generate PV or PVC YAML with `kubectl create`. The Persistent Volumes concept page,[^1] the PV task page[^5] and the StorageClass page[^2] have examples to start from.
- **Use `kubectl explain`** when you forget a field: `kubectl explain pv.spec.nodeAffinity --recursive`, `kubectl explain pvc.spec`.
- **Check names and namespaces.** A claim in the wrong namespace, or a misspelt `claimName` or `storageClassName`, causes most "it doesn't work" moments.
- **Verify.** After creating a claim, `kubectl get pvc` should show `Bound` (or `Pending` for a `WaitForFirstConsumer` class until the Pod exists). After the Pod starts, `kubectl exec` into it and write a file to the mount path. Delete the Pod, let it come back, and check the file is still there.
- **Mind the units.** `1Gi` (gibibyte, 2³⁰ bytes) isn't `1G` (gigabyte, 10⁹ bytes).

## Cheat sheet

```text
Binding needs:       same class · modes offered · size ≥ request · same volumeMode
                     · selector matches · PV Available
storageClassName:    "x" = class x | "" = no class, never provisioned
                     | unset = default class (if one exists)
Access modes:        RWO = one node | ROX = many nodes, read | RWX = many nodes, write
                     | RWOP = one Pod (CSI only); modes don't enforce read-only
Reclaim:             Retain (manual PV default) | Delete (StorageClass default) | Recycle (deprecated)
Released PV reuse:   kubectl patch pv X --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'
Binding mode:        Immediate (default) | WaitForFirstConsumer (Pending until a Pod uses it;
                     never with nodeName)
local PV:            needs nodeAffinity; class provisioner kubernetes.io/no-provisioner
Default class:       annotation storageclass.kubernetes.io/is-default-class: "true"
Expand:              allowVolumeExpansion: true on the class, then edit the PVC; grow only
Claim + Pod:         same namespace; volumes[].persistentVolumeClaim.claimName
StatefulSet:         volumeClaimTemplates → claim <template>-<sts>-<ordinal> per replica;
                     persistentVolumeClaimRetentionPolicy whenDeleted/whenScaled: Retain | Delete
CSI driver:          controller (Deployment: driver + provisioner/attacher/resizer/snapshotter)
                     + node plugin (DaemonSet: driver + node-driver-registrar)
CSI objects:         CSIDriver · CSINode (per node) · VolumeAttachment (per attach)
Failure → place:     Pending PVC → provisioner | FailedAttachVolume → attacher
                     | FailedMount → node plugin + kubelet
Inline volumes:      ephemeral.volumeClaimTemplate (any provisioner; PVC <pod>-<vol>)
                     | csi: (Ephemeral-mode drivers only)
Snapshots:           CRDs + snapshot controller; restore via PVC dataSource
```

## References

[^1]: [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/), Kubernetes documentation (v1.37).
[^2]: [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/), Kubernetes documentation.
[^3]: [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/), Kubernetes documentation.
[^4]: [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/), Kubernetes documentation.
[^5]: [Configure a Pod to Use a PersistentVolume for Storage](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/), Kubernetes documentation.
[^6]: [Change the Reclaim Policy of a PersistentVolume](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/), Kubernetes documentation.
[^7]: [Change the default StorageClass](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/), Kubernetes documentation.
[^8]: [Container Storage Interface (CSI) for Kubernetes Goes Beta](https://kubernetes.io/blog/2018/04/10/container-storage-interface-beta/), Kubernetes blog.
[^9]: [Deploying a CSI Driver on Kubernetes](https://kubernetes-csi.github.io/docs/deploying.html), Kubernetes CSI Developer Documentation.
[^10]: [Kubernetes API: Storage resources](https://kubernetes.io/docs/reference/kubernetes-api/storage/), Kubernetes documentation: CSIDriver, CSINode, CSIStorageCapacity, VolumeAttachment.
[^11]: [Ephemeral Volumes](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/), Kubernetes documentation.
[^12]: [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/), Kubernetes documentation.
[^13]: [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), Kubernetes documentation.
[^14]: [Velero documentation](https://velero.io/docs/), Velero project.
