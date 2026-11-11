# Fast sbx sandbox creation on ZFS

Nov 7, 2026

## Problem and cause

Creating a Docker Sandboxes (sbx) VM is slow when containerd's erofs snapshot directory sits on a ZFS dataset. Block size=16k, disabling sync writes and disabling direct I/O on the dataset doesn't help.

`sbx diagnose` shows `zfs_dmu_offset_next_sync` must be 0 and instructs to do that system wide. If the system wide change is not desired, see below.

The snapshot directory is:

```sh
~/.local/state/sandboxes/sandboxes/sandboxd/containerd/root/io.containerd.snapshotter.v1.erofs/snapshots
```

Below it is called `<snapshots dir>`.

## Fix: ext4 on a zvol

The snapshots stay on the ZFS pool, but inside a zvol formatted as ext4, so they are no longer files on a ZFS filesystem. The volume name `rpool/sbx-snap` is an example.

1. Stop the daemon: `sbx daemon stop`.
2. Create and format the volume:

   ```sh
   sudo zfs create -V 40G -o volblocksize=16k -o compression=lz4 rpool/sbx-snap
   sudo mkfs.ext4 /dev/zvol/rpool/sbx-snap
   ```
3. Copy the existing snapshots into it. The trailing slashes matter: they also copy the directory's owner and permissions.

   ```bash
   sudo mount /dev/zvol/rpool/sbx-snap /mnt
   sudo rsync -aHAX <snapshots dir>/ /mnt/
   sudo umount /mnt
   ```
4. Mount it over the snapshots directory: `sudo mount /dev/zvol/rpool/sbx-snap <snapshots dir>`.

## Mount at boot and verify

Add one line to `/etc/fstab`. fstab doesn't expand `~`, so use the full path to your home directory. The options make systemd wait for the zvol and keep booting if it's missing.

```sh
/dev/zvol/rpool/sbx-snap  /home/<user>/.local/state/sandboxes/sandboxes/sandboxd/containerd/root/io.containerd.snapshotter.v1.erofs/snapshots  ext4  defaults,nofail,x-systemd.requires=zfs-volumes.target  0  2
```

Check that the mounted directory belongs to you, like its parent:

```sh
ls -ld <snapshots dir> <snapshots dir>/..
```

If it shows `root`, fix only the top directory, not recursively: `sudo chown "$(id -un)":"$(id -gn)" <snapshots dir>`. The `lost+found` directory belongs to root, which is normal.

Finally, start the daemon and verify.