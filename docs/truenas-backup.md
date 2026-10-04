Auto-generated - no human review yet
# TrueNAS storage and backup for the prod cluster
Manual setup outside Argo CD: the TrueNAS SCALE dataset behind the `truenas-nfs-*` storage classes
(`k8s-prod/apps/csi-nfs/storage-classes.yaml`) and its rsync backup to a Synology.

## Open steps
1. **Test a restore** (not done yet). Until this works, the backup is unverified, in particular whether the Synology's
   rsync accepts `--fake-super` (see [Restore test](#restore-test)).
2. **Move the existing immich PVCs** to the new storage classes. `storageClassName` of a PVC is immutable and a PV keeps
   the share and subdirectory it was provisioned with, so the old PVCs stay on `/mnt/hdd/general/k8s` until they are
   recreated (or their data is copied and bound to static PVs).
3. **Snapshot cron job** failed with exit status 1 when started via "Run Now". Likely cause: `snapshot-manager` has
   no `zfs allow` on the new dataset yet (see [Snapshot before rsync](#snapshot-before-rsync)).

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

Same setup as in [rsync.md](rsync.md#snapshot-before-rsync), with the `snapshot-manager` user:
```sh
# once, as root in System → Shell
zfs allow snapshot-manager mount,snapshot,destroy hdd/general/k8s-prod
```
Cron job running as `snapshot-manager`, 1 minute before the rsync task:
```sh
zfs destroy hdd/general/k8s-prod@rsync 2>/dev/null; zfs snapshot hdd/general/k8s-prod@rsync
```
Untick "Hide Standard Error" to see failures. Debug in System → Shell:
```sh
zfs allow hdd/general/k8s-prod
zfs list -t snapshot -r hdd/general/k8s-prod
```

## rsync to the Synology
General TrueNAS → Synology rsync setup (module mode, used for personal data): [rsync.md](rsync.md).
This dataset uses SSH mode instead, because it must keep file ownership: module mode is unencrypted, needs a password file, and the Synology daemon may refuse
`--fake-super`. The Synology's rsync cannot be configured, so `-M--fake-super` makes the remote rsync store
ownership in xattrs (`user.rsync.%stat`) instead of needing root.

Synology (DSM UI):
- Control Panel → Terminal & SNMP: SSH on. File Services → rsync: on. User & Group → Advanced: user home service on.
- User `truenas-backup` with read/write on the target share (DSM 7 limits SSH to the administrators group).
- TrueNAS's public key in `~truenas-backup/.ssh/authorized_keys` (home not group/world-writable, `.ssh` `700`,
  `authorized_keys` `600`).

TrueNAS:
- Credentials → Backup Credentials → SSH Connections: manual, user `truenas-backup`, new keypair, discover host key.
- Data Protection → Rsync Tasks:

| Field | Value |
|---|---|
| Path | `/mnt/hdd/general/k8s-prod/.zfs/snapshot/rsync/` |
| User | `root` |
| Direction | Push |
| Rsync Mode | SSH, connection from the keychain |
| Remote Path | `/volume1/<share>/<subdir>` |
| Archive, Preserve Permissions | on |
| Auxiliary Parameters | `-M--fake-super --numeric-ids` |

On the Synology all files belong to `truenas-backup`; real ownership only comes back through an rsync pull with
`--fake-super`, not via File Station or SMB.

## Restore test
Second rsync task: Direction **Pull**, same SSH connection, user `root`, same auxiliary parameters, Path a scratch
dataset such as `/mnt/hdd/general/restore-test`. Then compare `ls -ln` with the source. Matching UIDs/GIDs (e.g.
for the `immich--immich-media` directory) mean fake-super works; everything owned by root means the xattrs were not written
(check the task log for xattr errors).
