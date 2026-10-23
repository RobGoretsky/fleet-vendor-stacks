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
   checks all three paths' filesystem type (`stat -f -c %T` = `nfs`)
   before `immich-server` or `immich-machine-learning` are allowed to
   start (`depends_on: ... condition: service_completed_successfully`).
   Belt-and-braces on top of (1) -- if automount's blocking-first-access
   behavior ever doesn't fire the way expected, this still refuses to
   come up on a wrong path instead of writing to it. It checks fstype,
   not `mountpoint -q`: a bind mount is *always* reported as a mountpoint
   from inside the container, even when the host-side directory behind
   it is just an empty local dir with no NFS mounted -- `mountpoint -q`
   would pass in exactly the silent-data-loss case it's meant to catch.

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
`${APPDATA_ROOT}/secrets/services/immich-db-password`, a single-line file
rendered by fleet-deploy from dagu-dags `secrets/services/` (rotate with
`robapp secret rotate immich-db-password` plus an `ALTER USER` in Postgres;
see its `reset` entry in dagu-dags `secrets.yaml`). No password lives in this
repo.

## Image pins

Pinned by digest (fleet's no-`:latest` rule; multi-arch index digests),
resolved 2026-10-23:

| service | image:tag | digest |
|---|---|---|
| immich-server | `v3.3.5` | `sha256:485d546a2e31...4352a6` |
| immich-machine-learning | `v3.3.5` | `sha256:5b28c9352c83...a40f56` |
| postgres | `14-vectorchord0.4.3-pgvectors0.2.0` | `sha256:bcf6335719...b1c23` |
| redis (valkey) | `9` | `sha256:c123e3715db6...708e1d` |

Postgres and valkey match Immich's own v3.3.5 example compose
(`docker/docker-compose.yml`) digest for digest. v3.3.5 is the first release
whose ML image ships certifi 2026.10.19 (the eMudhra/emSign "ECC Root CA - G3"
distrust); verified in the pinned image.

**Upgrading.** Server and ML always move together, to the same tag. The
server runs its own schema migrations on first start against whatever is in
Postgres, and they are one-way -- an older image will not start on a migrated
DB, so rollback is a restore of a pre-upgrade dump, not a tag revert. Before
merging a bump: take a fresh dump (below), update the phones' Immich apps
first on a major bump (the server only supports clients of its own major
version), and read the release notes for every major in between.

```
ssh robstinybox "sudo -u svc bash -c 'docker exec -i immich_postgres pg_dumpall --clean --if-exists -U immich | gzip > /home/svc/appdata/immich/immich-dump-pre-upgrade-\$(date +%F).sql.gz'"
```

That lands next to the live data on box NVMe; Immich's own nightly dumps in
the NFS library's `backups/` are the off-box copy.

The DB is on VectorChord (the `ghcr.io/immich-app/postgres` image, no
`DB_VECTOR_EXTENSION` set), so v3's removal of pgvecto.rs needs no action
here. Anything that queries Immich's Postgres directly (kitchen photos,
vacation-finder) is coupled to its schema and can break on any minor release.
