# Host: pbs.l — Proxmox Backup Server
# Access: ssh root@pbs.l  (root-only). Version: proxmox-backup-server 4.1.0-1
# Hostname: pbs — a dedicated PBS guest on pve.l.

## What this host does (backup strategy, corrected)

Earlier inventory assumed "no PBS and no syncoid" on the Proxmox host. That
was wrong. This host proves a two-layer backup strategy IS active:

1. **ZFS dataset replication (data)** — the `zpool_home/data` tree on pve.l
   is replicated to this host under `pbs/data_bkp` (same child dataset names:
   photos, videos, nextcloud, dropbox, git, music, documents, programs, src,
   iso). The received datasets carry `autosnap_*` snapshots (weekly/monthly/
   yearly), i.e. sanoid on pve.l snapshots and a ZFS send/recv replicator
   (syncoid-style) pushes them here. ~16 snapshots per dataset retained.
2. **VM/CT backups (vzdump → PBS)** — pve.l runs nightly backups (observed
   23:00 UTC) of every VM/CT into the `pbs-vms` datastore under namespace
   `pve-vms`. VM groups: 100–104, 109, 120, 9000 (template); CT groups:
   105, 110, 111. ~23 nightly snapshots retained per guest.

Layering note: ZFS replication protects the shared data tree; PBS protects
the guests themselves. Two independent mechanisms, both reaching pbs.l.

## Datastore

- `pbs-vms` — path `/pbs/vms_bkp`, ZFS-backed (pool `pbs`), 1.8 TiB pool,
  ~61% allocated. ~416 GB of dedup'd chunks for ~23 VM/CT snapshot sets.
- Namespace: `pve-vms` (single). Datastore-admin ACL on it for `pve@pbs`.

## Jobs / schedules

- **Backup** (client side, pve.l): nightly ~23:00 UTC, all VMs/CTs.
- **Verify**: `v-e17cdca4-f9de` — Fri 14:00, `ignore-verified`, verify backups
  older than 30 days.
- **Prune**: `default-pbs-vms-235250fe-d714-4a` — daily 14:00, store `pbs-vms`
  (no custom keep-* in config; retention driven by the backup job on pve.l,
  observed ~23 nightly snapshots kept).
- **Garbage collection**: Sat 18:15 (last run OK, removes orphaned chunks).

## Users / auth (names only — credentials redacted)

- `root@pam` — admin login (email redacted to admin@example.com).
- `pve@pbs` — service user with two API tokens:
  - `pve-token` — DatastoreAdmin on `/datastore/pbs-vms` (used by pve.l for
    VM/CT backup push).
  - `zfs-token` — used for the data-replication path.
  Both tokens exist; token secrets are NOT stored in this repo.

## Notifications

- Webhook to Telegram for job/GC notifications (endpoint + bot token +
  chat id are live credentials — intentionally not reproduced here).
- Default "mail to root" matcher exists but is disabled.

## Sanitized config examples

- `datastore.cfg` — datastore + GC schedule (values as-is, no secrets).
- `verification.cfg` — weekly verify job (no secrets).
- `prune.cfg` — daily prune job (no secrets).

Files under `/etc/proxmox-backup/` that are NOT reproduced: `user.cfg`
(password hashes / token secrets), `token.shadow`, `shadow.json`
(password hashes), `authkey.key`, `proxy.key`, `csrf.key`,
`notifications.cfg` (contains the live Telegram bot token + chat id),
`acl.cfg` (token names only, already noted above).
