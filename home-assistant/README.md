# home-assistant

Household smart-home stack: Home Assistant core, a loopback-only mosquitto
MQTT broker, and ring-mqtt (Ring integration). Deployed on **robstinybox**
by the unified engine off this repo's `compose.yaml` -- see the header
comment there for the per-service rationale (host network for mDNS/SSDP
discovery, MQTT on localhost).

This is a one-time **migration from the NAS**, not a fresh install -- HA's
config (including its git history) moves over as-is. Full cutover runbook:
Obsidian `Tech-Learning/Wave3-HA-Cutover`. This file covers only the
bring-up mechanics that live with the stack.

## Pre-cutover: directory + copy

Run on the box, as `svc` (or via `sudo -u svc`):

```
mkdir -p /home/svc/appdata/homeassistant /home/svc/appdata/ring-mqtt /home/svc/appdata/mosquitto
```

Then rsync each NAS config dir over (commands + exact flags in the
runbook's §C -- this is done from the box, over the NAS's SSH port 7600,
excluding HA's `backups/` and rotated logs, which are handled separately
below), followed by `chown -R svc:svc` on both destination trees.

## Backups on NFS

HA's `Settings > Backups` archives move from the NAS's local
`backups/` dir to `${MEDIA_ROOT}/backups/home-assistant` (NFS-mounted
`/mnt/nasdata` on the box) -- redundant NAS disk instead of the box's
single NVMe. **Before relying on this**, the runbook's root-write probe
must pass (`docker run --rm -v .../backups/home-assistant:/b alpine:3.20
touch /b/.probe`); if it fails with permission denied, the NFS export's
root-squash setting needs a DSM-side fix first. Do this probe before the
cutover, not after -- HA writing its first backup post-cutover is the
first time this path gets exercised for real.

## Config edits required before the DNS flip

The copied `configuration.yaml` `http:` block and `.storage/core.config`
still point at the NAS (`trusted_proxies`, `internal_url`/`external_url`).
Both need editing to the box's addresses and the new `homeassistant.rig`
name **before** traffic is flipped to it -- exact values are in the
runbook, not duplicated here since they're one-time edits, not stack
config.

## Git remote

The config dir is a `homeassistant-config` git checkout. The box's `svc`
user has an HTTPS GitHub credential helper but no SSH key, so the
migrated checkout's remote must be switched from
`git@github.com:...` to `https://github.com/RobGoretsky/homeassistant-config.git`
as part of the copy (runbook §C). The nightly autocommit DAG moves to
`dags-robstinybox/ha-git-backup.yaml` at the same time.

## Rollback

`docker start` the NAS's original `homeassistant`/`ring-mqtt`/
`eclipse-mosquitto` containers (stopped, never removed, at cutover) and
flip DNS back. Full steps: runbook §E.
