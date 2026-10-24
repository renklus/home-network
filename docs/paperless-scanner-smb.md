# Scanner → Paperless consume folder over SMB
Manual TrueNAS setup outside Argo CD: the network scanner writes over SMB into Paperless' consume folder, which the
paperless pod (`k8s-prod/apps/paperless/`) mounts over NFS and polls every 30 s.

Paperless uses UID/GID 3003:3003, the k8s user shared by all apps
Scanner uses TrueNAS user with UID `3004`

The pods and the scanner don't share a group. A POSIX ACL on the consume folder gives the scanner access to exactly
that folder, and its default entries give Paperless access to every file the scanner creates.

## Background: POSIX ACLs
The datasets use the "Generic" preset, so their ACL type is POSIX (`POSIX1E`), even where the UI only shows Unix
permissions. An ACL extends owner/group/other with named `USER`/`GROUP` entries and a `MASK` that caps them. A
directory has two independent ACLs:
- Access ACL (`"default": false`): who may use the folder itself, e.g. create and delete files in it.
- Default ACL (`"default": true`): grants nothing on the folder; new files and subfolders receive it as their ACL.

TrueNAS SCALE has no `getfacl`/`setfacl`; use the middleware API (`midclt`), as below. The UI's ACL editor only
works on whole datasets.

## 1. Traverse on the parent folders
This is only necessary on parents folder with acltype 'NFS*' (check with `midclt call filesystem.getacl "/mnt/hdd/general/k8s-prod-no-backup/paperless--paperless-consume" | python3 -m json.tool`)
Samba enters the share as the scanner user, so UID 3004 needs execute (traverse) on NFS type parents (possibly none)
```sh
for p in /mnt/hdd /mnt/hdd/general /mnt/hdd/general/k8s-prod-no-backup; do
  echo "== $p"; midclt call filesystem.getacl "$p" true | python3 -m json.tool
done
```
- `acltype: POSIX1E` with mode `755`: other has `r-x`, nothing to do (`k8s-prod-no-backup` is set up this way).
- `acltype: NFS4`: Datasets → the dataset → Edit permissions → Add item: Who: User, the scanner user; Permissions:
  Basic, **Traverse**; Flags: Basic, **No inherit**. Save without "Apply permissions recursively".

## 2. ACL on the consume folder
The folder exists once the PVC is bound; csi-nfs creates it as root.
```sh
D=/mnt/hdd/general/k8s-prod-no-backup/paperless--paperless-consume
chown 3003:3003 "$D"
midclt call --job filesystem.setacl "$(cat <<EOF
{"path": "$D", "acltype": "POSIX1E", "dacl": [
  {"default": false, "tag": "USER_OBJ",  "id": -1,   "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": false, "tag": "GROUP_OBJ", "id": -1,   "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": false, "tag": "USER",      "id": 3004, "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": false, "tag": "MASK",      "id": -1,   "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": false, "tag": "OTHER",     "id": -1,   "perms": {"READ": false, "WRITE": false, "EXECUTE": false}},
  {"default": true,  "tag": "USER_OBJ",  "id": -1,   "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": true,  "tag": "GROUP_OBJ", "id": -1,   "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": true,  "tag": "USER",      "id": 3003, "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": true,  "tag": "USER",      "id": 3004, "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": true,  "tag": "MASK",      "id": -1,   "perms": {"READ": true,  "WRITE": true,  "EXECUTE": true}},
  {"default": true,  "tag": "OTHER",     "id": -1,   "perms": {"READ": false, "WRITE": false, "EXECUTE": false}}
]}
EOF
)"
midclt call filesystem.getacl "$D" | python3 -m json.tool
```
If `setacl` rejects a field, compare with the `getacl` output of step 1, which uses the same structure.

| Entry | Why |
|---|---|
| Access `USER_OBJ`/`GROUP_OBJ` 3003 | Paperless owns the folder: it may delete every file in it (deleting depends on the folder, not the file). No named `USER 3003` needed. Still works through the group if csi-nfs recreates the folder as root. |
| Access `USER 3004` | The scanner may create files. |
| Access `OTHER` none | Nobody else. |
| Default `USER 3003` | Scanned files are owned by 3004; Paperless needs a named entry to read them (and to delete inside subfolders the scanner creates). |
| Default `USER 3004` | The scanner keeps access to subfolders it creates. |
| `MASK` rwx | Caps all named entries and `GROUP_OBJ`; must not cut them. |

POSIX requires every ACL, access and default, to contain `USER_OBJ`, `GROUP_OBJ` and `OTHER`, and a `MASK` once it
has named entries, so these appear twice.

## 3. SMB share
Shares → SMB → Add:
- Path: `/mnt/hdd/general/k8s-prod-no-backup/paperless--paperless-consume` (the folder only, not the dataset).
- Purpose: Default share parameters. Leave the dataset on the Generic preset: switching it to SMB/Multiprotocol
  breaks `chown`/`chmod` for the pods (see [truenas-backup.md](truenas-backup.md#datasets)).
- Access: only the scanner user needs to connect; the filesystem ACL limits everyone else anyway.

Configure the scanner with this share and the scanner user's credentials.

## 4. Test
Scan a page. Paperless picks it up within about 35 s (30 s polling plus 5 s stability delay), imports it and removes it
from the folder. To inspect a file before it is consumed:
```sh
midclt call filesystem.getacl "$D/<file>" | python3 -m json.tool
```
It must have a `USER` entry for `3003` with `READ` and `WRITE`, and the `MASK` must allow both. Samba applies its
`create mask` with `chmod`, which on a file with an ACL lowers the mask and can silently cut the named entries. If the
mask is restricted, set `inherit acls = yes` in the share's auxiliary parameters and test again.

A file that stays in the folder: check the paperless pod log (`kubectl -n paperless logs deploy/paperless`) and the
file's ACL as above.

## Recreated PVC
With `Retain`, a new `paperless-consume` PVC reuses the same subfolder, so the ACL stays. If the folder itself is
deleted and recreated by csi-nfs, repeat step 2.
