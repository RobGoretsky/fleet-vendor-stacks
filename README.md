# fleet-vendor-stacks

The gated monorepo for every **no-build third-party compose stack** in the
fleet -- deploy glue (a compose, optionally a thin Dockerfile + entrypoint
over vendor code), one subdir per stack. If you'd open it in an IDE to add
features, it belongs in its own graduated repo instead; if it's a compose
over someone else's images, it belongs here
(Tech-Learning/Fleet-Unified-Deploy-Engine).

Each stack is deployed by the unified engine
(`dagu-dags/scripts/fleet_deploy.py`): its registry entry in
`dagu-dags/registry.yaml` carries a `deploy: {compose: <subdir>/compose.yaml}`
block (path relative to this repo's checkout on the target host), and the
engine runs `docker compose --env-file ~/fleet.env up -d --build` from the
compose's directory whenever the last commit touching it moves. Change
detection is per subdir -- a tweak to one stack never restarts its
neighbors. `fleet-deploy` clones this repo into every host's checkout
itself the moment a registry entry needs it there -- nothing to clone by
hand.

**Pinned-tags rule**: every upstream image pins an exact tag (never
`:latest`). A git merge is the only thing that changes what runs; an
update is a conscious PR that bumps the pin.

## Stacks

| Subdir | Host | What |
|---|---|---|
| `arr-stack/` | robstinybox | The *arr media-automation fleet (sabnzbd, prowlarr, sonarr, radarr, lazylibrarian, audiobookshelf, seerr) -- SQLite configs on the box NVMe (`${APPDATA_ROOT}`), media over NFS (`${MEDIA_ROOT}`). Cutover runbook: Tech-Learning/Wave1-arr-Cutover. |
| `docker-ro-proxy/` | nas + robstinybox | GET-only docker socket proxy so the fleet cockpit can read container state on every fleet host, not just the one it runs on. Design: Tech-Learning/Fleet-Cockpit-Multihost-Docker. |
| `home-assistant/` | robstinybox | Home Assistant core + mosquitto (MQTT, loopback-only) + ring-mqtt, migrated from the NAS. Config is a git checkout on `${APPDATA_ROOT}`, backups on NFS `${MEDIA_ROOT}`. Cutover runbook: Tech-Learning/Wave3-HA-Cutover. |
| `adguard-home/` | nas | Fleet DNS server, host network (deletes the docker-proxy UDP hairpin so containers can resolve fleet names). Cutover runbook: Tech-Learning/Wave3-HA-Cutover §B. |

## Host checkout

`fleet-deploy` clones this repo into a host's `$MAINLINE_ROOT` itself the
first time a registry entry on that host needs it, and keeps it current
on every later run (ff-only pull) -- there's no per-host checkout step or
`checkout.<host>` field to maintain by hand.
