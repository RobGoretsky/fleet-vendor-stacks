# immich

Family photo library: server, ML, Postgres, and Redis, deployed on
**robstinybox** by the unified engine off this dir's `compose.yaml`. Wave 8
of the fleet migration -- read that file's header for the per-setting
rationale (why the mount-guard container exists, why `DB_STORAGE_TYPE`
flips to `SSD`). Full cutover runbook: Obsidian
`Tech-Learning/Wave8-Immich-Cutover`.

## The storage split (ruling 7)

Compute moves to the box's NVMe; **uploads and both external photo
libraries stay on the NAS**, reached over NFS -- there's no redundant disk
on the box yet, and 55+ GB of photos has nowhere else to safely live.
`immich.nas` -> `immich.rig`.

| What | Where | How |
|---|---|---|
| Postgres, Redis, ML model cache | box NVMe, `${APPDATA_ROOT}/immich/{postgres,redis,model-cache}` | local bind mount |
| Upload library (55 GB, rw) | NAS `/volume1/docker/immich/library` | NFS, `no_root_squash` |
| Rob's external library (ro) | NAS `/volume1/photo` | NFS |
| Pam's external library (ro) | NAS `/volume1/homes/pam/Photos` | NFS |

## `/etc/fstab` on robstinybox (the ordering guarantee)

Immich has no built-in check that its data paths are real mounts before it
starts -- if compose comes up before NFS does, it silently creates empty
local directories and serves an empty library (see compose.yaml's header
comment for the full failure mode). Two defenses:

1. **`x-systemd.automount`** on each fstab line below. The mount unit
   exists at boot but doesn't actually mount until first access; that
   first access **blocks** until the mount completes (or
   `x-systemd.mount-timeout` expires and the access fails loudly). Compose
   binding these paths into containers is that first access, so it can't
   silently race an unmounted NFS export the way a plain `_netdev` fstab
   entry (wave 1's `/mnt/nasdata` convention) can on a slow boot.
2. **`mount-guard`**, a one-shot init container in `compose.yaml` that
   runs `mountpoint -q` against all three paths before `immich-server` or
   `immich-machine-learning` are allowed to start (`depends_on: ...
   condition: service_completed_successfully`). Belt-and-braces on top of
   (1) -- if automount's blocking-first-access behavior ever doesn't fire
   the way expected, this still refuses to come up on a wrong path
   instead of writing to it.

Add to `/etc/fstab` (mirrors the `no_root_squash` / rw-vs-ro rows in
Obsidian `Tech-Learning/Phase2-Sitting-1` §1; NFSv4.1 + `hard`, same as
wave 1's `/mnt/nasdata` convention):

```
192.168.1.200:/volume1/docker/immich/library /mnt/immich-library nfs4 _netdev,x-systemd.automount,x-systemd.mount-timeout=30,hard,rw 0 0
192.168.1.200:/volume1/photo /mnt/immich-rob-photos nfs4 _netdev,x-systemd.automount,x-systemd.mount-timeout=30,hard,ro 0 0
192.168.1.200:/volume1/homes/pam/Photos /mnt/immich-pam-photos nfs4 _netdev,x-systemd.automount,x-systemd.mount-timeout=30,hard,ro 0 0
```

Then `sudo systemctl daemon-reload && sudo mount -a` and set in
`~/fleet.env`:

```
IMMICH_UPLOAD_MOUNT=/mnt/immich-library
IMMICH_ROB_PHOTOS_MOUNT=/mnt/immich-rob-photos
IMMICH_PAM_PHOTOS_MOUNT=/mnt/immich-pam-photos
```

These three fstab lines are the DSM-side NFS exports in
`Tech-Learning/Phase2-Sitting-1` §1 (rw/`no_root_squash` for the library,
ro for the two external libraries) -- they must exist on the NAS before
`mount -a` will succeed on the box.

## DB password

`POSTGRES_PASSWORD_FILE` / `DB_PASSWORD_FILE` (both natively supported --
Immich's own `_FILE`-suffix convention, verified against
docs.immich.app/install/environment-variables) point at
`/home/svc/.secrets/immich-db-password`, a single-line file with the same
password the NAS `.env` already has for `DB_PASSWORD` (copy it verbatim so
the restored dump's role password matches -- see the pg_dump/restore
section in `Tech-Learning/Phase2-Sitting-1` §3). No password lives in this
repo.

## Image pins

Pinned by digest (fleet's no-`:latest` rule), resolved 2026-09-10:

| service | image:tag | digest |
|---|---|---|
| immich-server | `v2.7.5` | `sha256:c15bff75068e...efaa5` |
| immich-machine-learning | `v2.7.5` | `sha256:a2501141440f...adf7f3` |
| postgres | `14-vectorchord0.4.3-pgvectors0.2.0` | `sha256:bcf6335719...b1c23` |
| redis (valkey) | `9` | `sha256:c123e3715db6...708e1d` |

Same tags as the NAS's current `.env` (server/ML floating on `release`,
NAS-pinned today at `2.7.5` per the wave 8 prep primer) -- **PIN v2.7.5**,
don't jump versions during a storage migration.
