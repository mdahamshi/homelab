# Proxmox VE host — storage & backup notes
# Host: pve.l   (single-node Proxmox VE)

## Guests (observed from host tap/veth interfaces)
- VMs: 100, 101, 102, 103, 104, 105, 109, 120 (+ 9000 template)
- LXC: 110
- Windows/misc on VLAN 800 (vmbr0.800, 10.0.8.0/24 placeholder)

## Storage
Single ZFS pool `zpool_home`, 7.27 TiB raw, ~12% allocated, healthy.

Datasets (names redacted to generic):
- `zpool_home/data`                     shared data root (recursively snapshotted)
- `zpool_home/data/photos`              352 GiB used — photo library (NFS -> home.l)
- `zpool_home/data/videos`              191 GiB used — video library (NFS -> home.l)
- `zpool_home/data/videos/movies`       92 GiB used — movies (excluded from autosnap)
- `zpool_home/data/nextcloud`           13 GiB used — Nextcloud data (NFS -> home.l)
- `zpool_home/data/dropbox`             Dropbox sync mirror (NFS -> home.l)
- `zpool_home/data/iso`                 39 GiB used — ISO images
- `zpool_home/data/src`                 9 GiB used — source tarballs
- `zpool_home/data/{git,music,documents,programs}` — misc data
- `zpool_home/bkp/vm`                   94 GiB used — VM/CT backup dumps

## Data sharing
The ZFS datasets above are exported over NFS v4 and mounted on the guest VMs
via autofs-style systemd automount (`_netdev,x-systemd.automount`).

## Backup strategy (layered) — corrected

> Earlier inventory claimed "no PBS and no syncoid on this host". That was
> wrong: pbs.l (Proxmox Backup Server, root@pbs.l) is actively receiving
> both layers described below. See `configs/pbs/notes.md`.

1. **ZFS snapshots + replication** — sanoid on this host snapshots
   `zpool_home/data` (cron every 15 min, policy `template_backup_light`:
   weekly x1, monthly x5, yearly x1; `data/videos/movies` excluded) and a
   syncoid-style ZFS send/recv replicates the tree to `pbs.l:pbs/data_bkp`
   (same child datasets, ~16 snapshots retained there).
2. **VM/CT backups → PBS** — nightly (~23:00 UTC) `vzdump` backups of all
   VMs/CTs into the `pbs-vms` datastore on pbs.l (namespace `pve-vms`),
   pushed with the `pve@pbs` service token. Verify weekly (Fri), prune
   daily (14:00), GC Sat.
   - A local dump dataset `zpool_home/bkp/vm` also exists (94 GiB used) —
     legacy/local dumps; the live target is PBS.

## Networking
- vmbr0: 10.0.0.1/24 placeholder (LAN bridge)
- vmbr0.800: VLAN 800 segment (10.0.8.0/24 placeholder)
- tailscale0: site-to-site Tailscale
- Services: PVE web UI (8006), SSH (22), NFS (2049), rpcbind (111),
  postfix (25), plus tailscale listeners.

## Time-based maintenance
- ZFS TRIM first Sunday of month, scrub second Sunday of month (zfs-linux cron).
