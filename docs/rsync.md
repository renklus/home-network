Auto-generated - limited human review
# rsync from TrueNAS to the Synology
General setup for pushing a TrueNAS dataset to the Synology (`app01l.dev.renklus.ch`) in rsync **module mode**, as
used for `hdd/general/k8s-prod` (see [truenas-backup.md](truenas-backup.md)). Module mode runs as a non-root TrueNAS user and does not keep file ownership: restored files belong to the task user and need a `chown` (see
[Restore](#restore)). `--fake-super` does not help: the Synology's rsync daemon treats it as `--super` unless the module
sets `fake super = yes` in `rsyncd.conf`, which DSM does not offer. Every file to back up must be readable by that
user, otherwise rsync skips it (exit code 23).

Source: https://www.reddit.com/r/truenas/comments/xk5nxm/solved_rsync_task_to_synology_nas/

## Commands
```sh
# list modules (= Synology shared folders)
rsync --password-file=~/rsync.secret <user>@<FQDN>::
# list remote files
rsync --password-file=~/rsync.secret <user>@<FQDN>::<module-name>/<remote-folder>/
# push (trailing slash on the source copies its contents, not the folder itself)
rsync -a --password-file=~/rsync.secret ./<local-folder>/ <user>@<FQDN>::<module-name>/<remote-folder>/
```

## Synology
- Create a user with **Full Control** (not just write permission) on the target folder. rsync also sets permissions
  and timestamps, which Synology's ACLs only allow with Full Control.
- File Services → rsync: enable the rsync service and the rsync account.
- File Services → rsync → Edit rsync Account: select the new user and set the password used by rsync.

## TrueNAS
- Create a user with read access to the data to back up.
- Write the Synology rsync password into a file in that user's home, e.g. `rsync.secret`. It must be owned by that user
  with mode `600`; rsync refuses a password file that others can read.
- Data Protection → Rsync Tasks:
  - Path: the snapshot, `/mnt/<pool>/<dataset>/.zfs/snapshot/rsync/.` (see below)
  - User: the TrueNAS user with read access
  - Direction: Push
  - Rsync Mode: Module
  - Remote Host: `app03l@app01l.dev.renklus.ch` (Synology user before the `@`)
  - Remote Module Name: Synology shared folder name, optionally with a subfolder (`<share>/<folder>`)
  - Auxiliary Parameters: `--password-file=/home/<user>/rsync.secret --no-owner --no-group`. The task has no
    password field. Archive (`-a`) includes `-o -g`, which the non-root daemon cannot apply (`chgrp ... Operation not
    permitted`, exit code 23 on every run, hiding real skips); unchecking "Preserve Permissions" only drops `-p`.

## Snapshot before rsync
rsync reads from a fixed-name snapshot `@rsync`, so it copies a consistent state instead of files that change during
the transfer.

- User `snapshot-manager`, with ZFS permissions delegated **per dataset** (run as root in System → Shell). `mount` is
  required by `snapshot` and `destroy`:
  ```sh
  zfs allow snapshot-manager mount,snapshot,destroy hdd/general/k8s-prod
  # check
  zfs allow hdd/general/k8s-prod
  ```
  Every new dataset needs its own `zfs allow`, otherwise the cron job fails with exit status 1.
- Cron job running as `snapshot-manager`, 1 minute before the rsync task:
  ```sh
  zfs destroy hdd/general/k8s-prod@rsync && zfs snapshot hdd/general/k8s-prod@rsync
  ```
  The destroy command will fail when the snapshot does not yet exist. Manually create the snapshot first.

TrueNAS roles (Credentials → Roles, e.g. snapshot-create/read/delete) do **not** work for this: they grant access to the
TrueNAS UI/API, not to the `zfs` command in a cron job.

## Restore
Second rsync task like the backup task, but Direction **Pull** and Path a scratch dataset such as
`/mnt/hdd/general/restore-test` (the task user needs write access). The restored files belong to the task user, so
note the original owner (`ls -ln`) and fix it afterwards as root, e.g. `chown -R <uid>:<gid> <dir>`.
