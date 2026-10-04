Auto-generated - no human review yet
# TrueNAS storage and backup for the prod cluster
Manual setup outside Argo CD: the TrueNAS SCALE dataset behind the `truenas-nfs-*` storage classes
(`k8s-prod/apps/csi-nfs/storage-classes.yaml`) and its rsync backup to a Synology.

## Open steps
1. **Move the existing immich PVCs** to the new storage classes. `storageClassName` of a PVC is immutable and a PV keeps
   the share and subdirectory it was provisioned with, so the old PVCs stay on `/mnt/hdd/general/k8s` until they are
   recreated (or their data is copied and bound to static PVs).

## Storage classes
Each PVC gets the subdirectory `<ns>--<pvc>` on the class's share.

| Class | Dataset | Backup | reclaimPolicy | Use for |
|---|---|---|---|---|
| `truenas-nfs` | `hdd/general/k8s-prod` | yes | Retain | data that exists nowhere else (photos) |
| `truenas-nfs-no-backup` | `hdd/general/k8s-prod-no-backup` | no | Retain | data with its own backup (the CNPG database: Immich's dump lives on the media PVC) |
| `truenas-nfs-temp` | `hdd/general/k8s-prod-no-backup` | no | Delete | caches that can be rebuilt (ML model cache) |

Each class has a `-soft` variant. `hard` blocks I/O until the NFS server is back; `soft` returns `EIO` after a timeout,
which can lose or corrupt writes, so use it only when the workload tolerates I/O errors.

With `Retain`, deleting a PVC leaves the PV `Released` and the subdirectory on the share. A new PVC with the same name
and namespace provisions into the same subdirectory again. Clean up by hand: `k8s01d delete pv <pv>`, then remove
the subdirectory on TrueNAS.

## Datasets
`hdd/general/k8s-prod` and `hdd/general/k8s-prod-no-backup` (a sibling, not a child, so it is not in the snapshot
below), both with:
- Dataset preset: **Generic** (POSIX ACLs). Not SMB/Multiprotocol: pods and kubelet (`fsGroup`) need plain
  `chown`/`chmod`, and Postgres refuses a data directory that is not `0700`/`0750`.
- Sync: Standard (Postgres runs on it), compression LZ4, atime off, snapdir hidden.
- Owner `root:root`, mode `755`, no ACL entries. The per-PVC subdirectories get their ownership from the pods.

## NFS shares
One share per dataset, both with:
- Path: the dataset's mount point. Networks/Hosts: only the prod nodes.
- Maproot User `root`, Maproot Group `root` (no Mapall): the csi-nfs controller creates and deletes the
  `<ns>--<pvc>` subdirectories as root, and kubelet applies `fsGroup`.
- Services → NFS: NFSv4 enabled (storage classes mount with `nfsvers=4.2`). SCALE uses numeric UIDs/GIDs with
  `sec=sys` by default; the "NFSv3 ownership model" option only exists on TrueNAS CORE.

Check after the first PVC exists: `ls -ln` shows the same numbers on TrueNAS and on the node. If the node shows
`nobody`/`4294967294`, check that `/sys/module/nfs/parameters/nfs4_disable_idmapping` is `Y` there.

## Snapshot before rsync
rsync reads from a ZFS snapshot of `hdd/general/k8s-prod` instead of the live dataset, so all files are from the same
moment. The CNPG database is on `k8s-prod-no-backup` and not part of it; its backup is Immich's own database dump
(daily by default) on the media PVC.

Snapshot cron job (`snapshot-manager`, `zfs allow` on `hdd/general/k8s-prod`) as in
[rsync.md](rsync.md#snapshot-before-rsync).

## rsync to the Synology
Module mode with a non-root TrueNAS user, as in [rsync.md](rsync.md), with Path
`/mnt/hdd/general/k8s-prod/.zfs/snapshot/rsync/.`. The backup does not keep file ownership. That is acceptable because
the dataset only holds media (a few owners, easy to `chown` back) and the database is backed up by Immich's dump.
A restore was tested: files come back, owned by the task user.

When adding a PVC on `truenas-nfs`, check that the backup user can read it. An app that writes `0700` directories or
`0600` files (like Postgres) is skipped by rsync (exit code 23); put such data on `truenas-nfs-no-backup` with its own
backup instead.

## Restore
As in [rsync.md](rsync.md#restore), then restore the owner of each PVC directory before the pod starts again, e.g.
for Immich (check the UID/GID on the source with `ls -ln` first):
```sh
chown -R <uid>:<gid> /mnt/hdd/general/k8s-prod/immich--immich-media
```
