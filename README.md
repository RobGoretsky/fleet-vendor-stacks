# fleet-vendor-stacks

The gated monorepo for every **no-build third-party compose stack** in the
fleet -- deploy glue (a compose, optionally a thin Dockerfile + entrypoint
over vendor code), one subdir per stack. If you'd open it in an IDE to add
features, it belongs in its own graduated repo instead; if it's a compose
over someone else's images, it belongs here
(Tech-Learning/Fleet-Unified-Deploy-Engine).

Each stack is deployed by the unified engine
(`dagu-dags/scripts/fleet_deploy.py`): its registry entry in
`dagu-dags/registry.yaml` carries `checkout.<host>` = this repo's checkout
path plus a `deploy: {stack_dir: <subdir>}` block, and the engine runs
`docker compose --env-file ~/fleet.env up -d --build` from the subdir
whenever the last commit touching it moves. Change detection is per
subdir -- a tweak to one stack never restarts its neighbors.

**Pinned-tags rule**: every upstream image pins an exact tag (never
`:latest`). A git merge is the only thing that changes what runs; an
update is a conscious PR that bumps the pin.

## Stacks

| Subdir | Host | What |
|---|---|---|
| `arr-stack/` | robstinybox | The *arr media-automation fleet (sabnzbd, prowlarr, sonarr, radarr, lazylibrarian, audiobookshelf, seerr) -- SQLite configs on the box NVMe (`${APPDATA_ROOT}`), media over NFS (`${MEDIA_ROOT}`). Cutover runbook: Tech-Learning/Wave1-arr-Cutover. |

## Host checkout

Cloned once by hand per host (selfsync never clones), then selfsync keeps
it current: on robstinybox,
`sudo -u svc git clone http://forgejo.nas/robg/fleet-vendor-stacks.git /home/svc/mainline/fleet-vendor-stacks`.
