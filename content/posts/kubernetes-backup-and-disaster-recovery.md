---
title: "Kubernetes Backup and Disaster Recovery"
date: 2026-11-17
draft: true
description: "How to back up a Kubernetes cluster and get it back after a disaster: RPO and RTO, where cluster state lives, rebuilding from Git, etcd snapshots and restores, Velero for objects and volumes, application-consistent database backups, immutable off-site storage and restore drills."
summary: "High availability protects you from a dead node. It does nothing for a deleted namespace, a corrupted etcd, a ransomware attack or a lost cloud account. Here is what to back up in a Kubernetes cluster, the tools that do it, and how to make sure the restore actually works."
tags: ["kubernetes", "backup", "disaster recovery", "etcd", "velero", "storage"]
categories: ["Programming"]
---

Kubernetes is good at surviving small failures. A container crashes and the kubelet restarts it. A node dies and its Pods are rescheduled elsewhere. Run three control plane nodes and you can lose one without noticing. It's easy to conclude that a cluster with enough replicas looks after itself.

It doesn't. High availability protects you from **losing a copy**. It does nothing when the copies themselves are wrong. Run `kubectl delete namespace production` and every replica of every object in it is gone within seconds, faithfully replicated across all three etcd members. A bad Helm upgrade, a corrupted etcd, a compromised account that deletes your cloud resources, or a whole region going offline are all the same kind of problem: what you need isn't another replica but a copy from *before* things went wrong, kept somewhere the disaster couldn't reach.

That's what backup and disaster recovery are about. This post covers what state a cluster holds and where, the three layers that protect it (Git, etcd snapshots and Velero), how to make database backups consistent, where to keep backups, and how to know your restore works before you need it.

The examples match Kubernetes v1.37, etcd v3.6 and current Velero.

## Start with two numbers

Before choosing tools, decide what "recovered" means for each workload. Two numbers capture it:

- **RPO (Recovery Point Objective):** how much data you can afford to lose, measured in time. An RPO of 24 hours means a nightly backup is enough. An RPO of five minutes means nightly snapshots won't do; you need something continuous, like database log shipping.
- **RTO (Recovery Time Objective):** how long you can afford to be down. An RTO of a day allows a manual rebuild. An RTO of fifteen minutes needs a standby cluster that's already running.

The numbers differ by workload. A stateless frontend has no RPO at all (there's no data to lose) and a short RTO (redeploy it). A billing database might need an RPO of seconds. A build cache needs neither: if it's gone, it gets rebuilt. Writing these down per application keeps you from protecting everything at the level of your most critical system, or worse, protecting your most critical system at the level of everything else.

## Where the state lives

A Kubernetes cluster spreads its state across more places than people expect:

| State | Where it lives | Lost if you lose… |
|---|---|---|
| API objects: Deployments, Services, ConfigMaps, Secrets, CRDs and custom resources, RBAC | etcd | the etcd data |
| Application data | PersistentVolumes, i.e. cloud disks, NFS, Ceph and so on | the storage backend |
| Cluster identity: CA and certificates, kubeconfigs, static Pod manifests, encryption config | `/etc/kubernetes/` on control plane nodes (self-managed clusters) | the control plane nodes |
| Manifests, Helm charts, values | Git, ideally | the Git server |
| External dependencies: load balancers, DNS records, IAM roles, managed databases, object storage | the cloud provider | the cloud account |

Two things stand out. First, **etcd holds objects, not data**. An etcd snapshot restores a PersistentVolumeClaim, but not a single byte of what was on the volume. Second, plenty of what makes your application work lives **outside** the cluster entirely. No Kubernetes backup tool restores a deleted Route 53 zone or a managed PostgreSQL instance; those need their own backups, usually through infrastructure as code and the provider's own snapshot features.

So a complete strategy has layers, each covering a different row of that table.

## Layer 1: rebuild from Git

The cheapest backup is not needing one. If every manifest, chart and values file lives in Git and a GitOps tool like Argo CD or Flux applies them, a lost cluster is "create a new cluster, point the GitOps tool at the repository, wait". The cluster infrastructure itself, its nodes, networks and IAM, should be code too.[^1]

This is the right recovery path for most stateless workloads, and it has a big advantage over restoring a backup: you get a clean cluster with exactly what's declared, with none of the manual changes, half-finished experiments or compromised objects an attacker may have left behind.

But Git doesn't hold everything:

- **Secrets.** You shouldn't keep plain Secrets in Git. If you use Sealed Secrets, the controller's private key must be backed up separately, or nothing in Git can be decrypted again. If you use External Secrets Operator, the secrets live in an external vault that needs its own backup.[^2]
- **State created at runtime.** Objects that controllers or people create: certificates issued by cert-manager, a namespace someone made by hand, an operator's custom resources, Jobs and their history.
- **Persistent data.** Git knows a PVC exists. It knows nothing about what's on it.

So Git is the foundation, and the next two layers fill the gaps.

## Layer 2: etcd snapshots

etcd is the cluster's database. Every object the API server knows about is stored there, so a snapshot of etcd is a complete, point-in-time copy of the cluster's API state. On a **self-managed** cluster (kubeadm, k3s, RKE2, bare metal), this is the most important backup you can take. On a **managed** cluster (EKS, GKE, AKS), you don't have access to etcd at all: the provider runs and backs it up, and you rely on layers 1 and 3 instead.

### Taking a snapshot

`etcdctl snapshot save` streams a consistent snapshot from a running member. It needs the etcd client certificates, which on a kubeadm cluster live in `/etc/kubernetes/pki/etcd/`:[^3]

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/backups/etcd-$(date +%Y%m%d-%H%M).db
```

If `etcdctl` isn't installed on the host, run it inside the etcd Pod, as shown in [the kubeadm upgrade post](/posts/kubeadm-cluster-upgrade/). Then check the file:

```bash
etcdutl snapshot status /var/backups/etcd-20261117-0300.db --write-out=table
```

`status` reports the snapshot's hash, revision, number of keys and size. A snapshot that won't open here won't restore either. Since etcd v3.5, the offline operations (`status` and `restore`) belong to `etcdutl`; the `etcdctl` versions are deprecated.[^4]

A few things to get right:

- **Snapshot one member, not all.** In a three-member cluster, every member holds the same data. One snapshot is enough; the restore recreates all members from it.
- **Automate it.** A CronJob or a systemd timer on a control plane node, every hour or every few hours depending on your RPO, with old snapshots pruned.
- **Get it off the node.** A snapshot sitting in `/var/backups` on the control plane node dies with that node. Copy it to object storage immediately (more on where below).
- **Back up `/etc/kubernetes/` too.** The cluster's CA and certificates aren't in etcd. Without the original CA, a restored cluster issues new certificates, and every kubeconfig and every node's kubelet stops trusting it.
- **Treat it as a secret.** An etcd snapshot contains every Secret in the cluster. If you've enabled [encryption at rest](/posts/kubernetes-secrets/), the Secrets in the snapshot are encrypted, which is good, but then you also need the `EncryptionConfiguration` file or access to the KMS key. A snapshot whose Secrets you can't decrypt restores a cluster in which no workload can read its credentials.

### Restoring a snapshot

Restoring etcd is a cluster-wide, all-or-nothing operation. It takes **every** object back to the moment of the snapshot. Anything created since is gone, and anything deleted since is back. It's the right tool when etcd itself is lost or corrupted, or the whole cluster's state is wrong. It's the wrong tool for "someone deleted one Deployment", which layer 3 handles much better.

On a kubeadm cluster with stacked etcd, the procedure on a single control plane node looks like this:[^3][^4]

```bash
# 1. Stop the API server and etcd by moving their static Pod manifests away.
sudo mkdir -p /etc/kubernetes/manifests-off
sudo mv /etc/kubernetes/manifests/{kube-apiserver,etcd}.yaml /etc/kubernetes/manifests-off/

# 2. Restore the snapshot into a NEW data directory.
sudo etcdutl snapshot restore /var/backups/etcd-20261117-0300.db \
  --data-dir /var/lib/etcd-restored

# 3. Point etcd at the new directory: in etcd.yaml, change the hostPath
#    of the etcd-data volume from /var/lib/etcd to /var/lib/etcd-restored.
sudo sed -i 's|path: /var/lib/etcd$|path: /var/lib/etcd-restored|' \
  /etc/kubernetes/manifests-off/etcd.yaml

# 4. Bring etcd and the API server back.
sudo mv /etc/kubernetes/manifests-off/*.yaml /etc/kubernetes/manifests/
```

Restoring into a new directory instead of overwriting `/var/lib/etcd` means you can go back if the snapshot turns out to be bad. Check the `sed` result before moving the manifests back: the data directory path appears in the manifest more than once (as a command flag and a volume), and only the volume's `hostPath` should change.

With **several** control plane nodes, every etcd member must be restored from the **same** snapshot file, each with its own `--name`, `--initial-advertise-peer-urls`, and the same `--initial-cluster` and a new `--initial-cluster-token`, so the restored members form a new cluster rather than trying to rejoin the old one.[^4] Stop all API servers first and start them only after all etcd members are back.

After the restore, expect some turbulence. Controllers wake up and compare the restored desired state with the real world. Pods that exist on nodes but not in the snapshot get deleted; Pods in the snapshot that no longer exist get recreated. Restarting the controller manager, scheduler and kubelets afterwards is a good idea, so none of them works from cached state.[^3] And the snapshot knows nothing about volume contents: if a disk was deleted after the snapshot, the PV object comes back, but points at a disk that no longer exists.

Restore to the **same** Kubernetes and etcd versions the snapshot was taken from. A snapshot is a copy of a database, not a portable export.

## Layer 3: Velero for objects and volumes

etcd snapshots are a blunt instrument, and unavailable on managed clusters. For everyday backup and restore, the standard open source tool is **Velero**. It works through the Kubernetes API instead of underneath it, so it runs the same way on any cluster, managed or not.

### How it works

Velero runs as a Deployment in the cluster. When you create a backup, it queries the API server for the objects you selected, writes them as JSON to a tarball in object storage (S3, GCS, Azure Blob or anything S3-compatible like MinIO), and, for the PersistentVolumes those objects use, either takes snapshots or copies the files. A restore does the reverse: it reads the tarball, recreates the objects through the API server in dependency order (namespaces and CRDs first, then the things inside them), and restores the volume data.[^5]

Because it goes through the API, Velero can do what an etcd snapshot can't:

- back up and restore **one namespace**, or only objects with a certain label
- restore into **a different cluster**, which makes it a migration tool as well
- restore into **a different namespace**
- run on **managed clusters**

Where backups go is configured by two custom resources: a `BackupStorageLocation` (the bucket for object tarballs) and optionally a `VolumeSnapshotLocation` (where native volume snapshots are taken). The provider plugin, e.g. for AWS, GCP or Azure, handles the cloud-specific parts.

### Installing

The CLI installs the server. For AWS, roughly:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:<version> \
  --bucket my-cluster-backups \
  --backup-location-config region=eu-central-1 \
  --snapshot-location-config region=eu-central-1 \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --features=EnableCSI
```

Pick the plugin version from the compatibility table in the Velero docs. `--use-node-agent` deploys a DaemonSet needed for file-level volume backups and data movement (below). Most teams install Velero with its Helm chart instead, which takes the same settings as values.

### Backups and schedules

A one-off backup of one namespace:

```bash
velero backup create shop-before-migration --include-namespaces shop --wait
velero backup describe shop-before-migration --details
velero backup logs shop-before-migration
```

For regular protection, create a schedule. It takes a cron expression and a TTL, after which Velero deletes the backup and its snapshots:

```bash
velero schedule create nightly \
  --schedule="0 3 * * *" \
  --exclude-namespaces kube-system,velero \
  --ttl 720h
```

That keeps thirty days of nightly backups of everything except system namespaces. For a stricter RPO on important namespaces, add a second, more frequent schedule just for them.

### Volume data: three ways

Velero backs up the object tarball the same way every time. For volume **data**, it has three methods, and choosing between them is the most important decision in a Velero setup:[^6][^7]

1. **CSI snapshots.** Velero asks the CSI driver for a `VolumeSnapshot` of each PVC, using the snapshot machinery described in [the storage post](/posts/persistent-storage-in-kubernetes/). Fast and cheap, but the snapshot stays **in the storage system**, usually in the same cloud account and region as the volume. Lose the account or the region and you lose the snapshots with it.
2. **CSI snapshot data movement.** Velero takes a CSI snapshot, mounts it, and uploads its contents to the backup bucket, then deletes the local snapshot. Enable it per backup with `--snapshot-move-data`. Slower, but the data now lives wherever your bucket lives, which can be another region or another provider. This is usually the right default for disaster recovery.
3. **File system backup.** The node agent reads the files straight from the volume mounted in the running Pod and uploads them with Kopia. It works with any volume type, even ones without snapshot support like `hostPath` or some NFS setups, but it isn't a point-in-time copy: files change while they're being read. Opt in per Pod with the annotation `backup.velero.io/backup-volumes: <volume-name>`, or for everything with `--default-volumes-to-fs-backup`.

Whichever you choose, the method doesn't make the data **consistent**. For that, you need to involve the application.

### Hooks: consistent snapshots

A snapshot of a disk under a running database captures whatever was on the disk at that instant, including half-written pages. Most databases recover from that, the way they recover from a power cut, but "most" and "usually" are not words you want in a recovery plan.

Velero **backup hooks** run commands inside a container before and after its volumes are backed up. Configure them with Pod annotations:[^8]

```yaml
metadata:
  annotations:
    pre.hook.backup.velero.io/container: mysql
    pre.hook.backup.velero.io/command: >-
      ["/bin/sh", "-c", "mysql -uroot -p\"$MYSQL_ROOT_PASSWORD\" -e 'FLUSH TABLES WITH READ LOCK; SELECT SLEEP(30);' &"]
    post.hook.backup.velero.io/container: mysql
    post.hook.backup.velero.io/command: >-
      ["/bin/sh", "-c", "mysql -uroot -p\"$MYSQL_ROOT_PASSWORD\" -e 'UNLOCK TABLES;'"]
```

For a filesystem, `fsfreeze` before and after does the same job for any application. Freezing pauses writes, so keep the window short: with CSI snapshots, the freeze only has to last until the snapshot is taken, which is usually seconds.

### Restoring

```bash
velero backup get
velero restore create --from-backup nightly-20261117030012
velero restore describe <restore-name> --details
```

Useful options:

- `--include-namespaces shop` restores just one namespace from a full backup.
- `--namespace-mappings shop:shop-restored` restores into a new namespace, so you can inspect the old state next to the current one.
- `--existing-resource-policy update`: by default, Velero **skips** objects that already exist in the cluster and only reports them as warnings. With `update`, it updates them to match the backup instead.[^9]

To move workloads to a new cluster, install Velero there, point it at the same bucket with the backup location set to read-only, so the new cluster can't delete or overwrite the old cluster's backups, and restore from it.

### Managed alternatives

The cloud providers offer managed backup services for their Kubernetes products, such as Backup for GKE.[^10] They do much the same job as Velero, back up to the provider's own storage and spare you running another component. The trade-off is that the backups live with the same provider, which matters for the scenarios below.

## Databases need their own backups

For a database, a volume backup, even a consistent one, is a coarse tool. It gives you the state at backup time, and nothing in between. If your RPO is minutes, nightly volume snapshots can't meet it.

Databases solve this themselves, and their native mechanisms are better than anything at the storage layer:

- **Logical dumps** like `pg_dump` or `mysqldump` produce a portable, version-independent export. Run them from a CronJob and upload the result to object storage. Simple and good for small databases, slow for large ones.
- **Continuous archiving** ships the database's write-ahead log (WAL in PostgreSQL, binlog in MySQL) to object storage as it's written, on top of periodic base backups. You can then restore to **any point in time**, e.g. one second before someone ran `DROP TABLE`. That gives an RPO of seconds.

On Kubernetes, database **operators** do this for you. CloudNativePG, for example, takes base backups and archives the WAL to object storage continuously, and can bootstrap a new cluster from that archive, at any recovery target time you choose.[^11] If you run a production database in Kubernetes, use an operator with this kind of backup support and treat Velero as a second line of defence for the database, not the first.

The same applies to other stateful systems with their own replication and backup tools: message brokers, search engines, object stores. Check what the system offers before relying on volume snapshots.

## Where backups live

A backup stored in the same place as the thing it protects will die in the same disaster. The old **3-2-1 rule** still applies: three copies of the data, on two different kinds of storage, with one copy off-site. For a cloud-hosted cluster, that translates to:

- **Another region.** A regional outage shouldn't take your backups with it. Replicate the backup bucket to a second region, or write to one there directly.
- **Another account.** If an attacker, or a broken automation, gets admin rights in your production account, they can delete the cluster *and* the backups. Keep backups in a separate account that production can write to but not delete from.
- **Immutability.** Ransomware goes after backups first. S3 Object Lock in compliance mode, and the equivalents on other providers, make objects undeletable and unmodifiable until a retention date, even for the account's root user.[^12] Velero, writing to a locked bucket, gets backups that nobody can delete early.
- **Encryption.** Backups contain every Secret in the cluster. Encrypt the bucket, restrict who can read it, and keep the encryption keys recoverable too: a key stored only in the destroyed account makes the backup useless.

## Disasters and what fixes them

Put together, each layer covers different scenarios:

| What happened | Recovery path |
|---|---|
| A Deployment or ConfigMap was deleted or broken | Re-sync from Git. Without GitOps, a Velero restore of just that object. |
| A namespace was deleted, including its volumes | Velero restore of the namespace, objects and volume data together |
| Someone corrupted rows in a database | The operator's point-in-time recovery, to just before the change |
| etcd is corrupted or lost on a self-managed cluster | Restore the etcd snapshot, plus `/etc/kubernetes/` if the nodes are gone |
| The whole cluster is gone | New cluster from infrastructure code, GitOps for workloads, Velero for the rest and the volumes |
| A region is down | The same as above, in another region, which only works if the backups and images are reachable from there |
| The account is compromised, ransomware | A new account, restore from immutable backups in a separate account, rebuild from Git, and rotate every credential |

Two details people miss in the bigger scenarios. Container **images** must be pullable in the recovery location: if your registry was in the lost region or account, your manifests point at images that no longer exist. And the cluster you restore into needs the same **CRDs, CSI drivers and StorageClasses**, or restored objects and volumes have nothing to bind to.

## Test the restore

A backup you've never restored is a hope, not a backup. Backup jobs fail quietly: a credential expires, a bucket policy changes, a new namespace isn't covered by the schedule, a hook breaks and the backup is "successful" but inconsistent. You find out during the disaster, which is the worst time.

So practise:

- **Monitor backups.** Velero exposes Prometheus metrics, including the time of the last successful backup per schedule. Alert when it's older than your RPO.
- **Restore regularly.** Spin up a scratch cluster, restore last night's backup into it, and check that the application starts and the data is there. Automate this if you can.
- **Run a drill.** At least once a year, play through a full disaster: the cluster is gone, start from nothing, with a stopwatch running. Your real RTO is whatever the stopwatch says, not what the design document says.
- **Write the runbook during the drill.** The person restoring at 3 a.m. may not be the person who set up the backups. Every step that needed tribal knowledge in the drill belongs in the runbook.

## Checklist

```text
Decide:     RPO and RTO per workload; not everything needs the same protection
Git:        all manifests, charts and infrastructure code; GitOps applies them
            + back up what Git can't hold (Sealed Secrets key, external vault)
etcd:       self-managed only; etcdctl snapshot save on a schedule, etcdutl snapshot status
            to verify, copy off the node, plus /etc/kubernetes/ and the encryption config
            restore = whole cluster back to snapshot time, same versions, new data dir
Velero:     scheduled backups with TTL; restore per namespace, into new namespaces,
            or into another cluster; existing objects are skipped unless told otherwise
Volumes:    CSI snapshot (fast, same account) | data movement (copies to bucket, DR)
            | file system backup (any volume, not point-in-time)
Consistent: hooks to flush and freeze; databases via their operator with PITR
Storage:    another region, another account, Object Lock, encrypted, keys recoverable
Outside:    images, DNS, IAM, managed databases need their own recovery path
Test:       alert on backup age, restore regularly, drill a full rebuild, keep a runbook
```

## References

[^1]: See [Infrastructure as Code](/posts/infrastructure-as-code/) on this blog for managing the cluster's own infrastructure as code.
[^2]: [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/), Kubernetes documentation. See also [Secrets in Kubernetes](/posts/kubernetes-secrets/) on this blog.
[^3]: [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/), Kubernetes documentation: backing up and restoring an etcd cluster.
[^4]: [Disaster recovery](https://etcd.io/docs/v3.6/op-guide/recovery/), etcd documentation (v3.6).
[^5]: [How Velero Works](https://velero.io/docs/main/how-velero-works/), Velero documentation.
[^6]: [CSI Snapshot Data Movement](https://velero.io/docs/main/csi-snapshot-data-movement/), Velero documentation.
[^7]: [File System Backup](https://velero.io/docs/main/file-system-backup/), Velero documentation.
[^8]: [Backup Hooks](https://velero.io/docs/main/backup-hooks/), Velero documentation.
[^9]: [Restore Reference](https://velero.io/docs/main/restore-reference/), Velero documentation.
[^10]: [About Backup for GKE](https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/concepts/backup-for-gke), Google Cloud documentation.
[^11]: [Backup](https://cloudnative-pg.io/documentation/current/backup/) and [Recovery](https://cloudnative-pg.io/documentation/current/recovery/), CloudNativePG documentation.
[^12]: [Locking objects with Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html), Amazon S3 User Guide.
