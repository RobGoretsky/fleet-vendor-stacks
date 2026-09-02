# docker-ro-proxy

A GET-only, read-only HAProxy proxy in front of the docker socket, deployed
on **every fleet host** (NAS = robsbigdisk, robstinybox = "the box") so the
fleet cockpit (`fleet-control-plane`) can list and inspect containers
fleet-wide, not just on whichever host it happens to run on.

## Why a separate proxy from crh's control proxy

`claude-remote-host` already runs a docker socket proxy
(`haproxy-docker-proxy.cfg`) that gives the claude container the power to
restart/stop/start/kill existing containers. That proxy is deliberately
**internal-only** (`claude-remote-host_docker-ctl` network, no published
port) because it grants write-ish power over the host's containers.

The cockpit only ever needs to *read* container state, and it needs to read
it from every host, including hosts it isn't running on -- so it needs a
proxy that is (a) GET-only, with zero write path, and (b) reachable over the
tailnet. Those are different trust levels and different exposure postures
from the control proxy, so this is a separate stack rather than widening the
control proxy's scope. See `haproxy-docker-ro.cfg`'s header for the full
allow/deny list, and Obsidian
`Tech-Learning/Fleet-Cockpit-Multihost-Docker` for the design.

## `~/fleet.env` variables

| Var | NAS (robsbigdisk) | robstinybox ("the box") |
|---|---|---|
| `DOCKER_GID` | `65536` | `989` |
| `DOCKER_RO_BIND_IP` | `127.0.0.1` | `192.168.1.201` |

**NAS quirk**: Tailscale on the NAS runs `--tun=userspace-networking`, so its
tailnet IP `100.93.26.74` is not a local interface and cannot be bound.
Inbound tailnet connections are delivered by `tailscaled` to
`127.0.0.1:2375`, so the NAS proxy binds loopback and is reachable from other
tailnet devices at `100.93.26.74:2375` -- but is **not** reachable on the LAN
IP `192.168.1.200`. And the same mode cannot dial OUT to tailnet IPs (verified: from the NAS,
`100.76.220.3` routes out eth0 and times out), so the box proxy binds its
**LAN IP `192.168.1.201`** -- the cockpit on the NAS reads it over the LAN,
the same way it already reads dagu and forgejo. A cockpit running on the box
would instead read the NAS over the tailnet (the box has a real `tailscale0`).

## Exposure posture

- GET-only at the proxy (see `haproxy-docker-ro.cfg`) -- no POST/PUT/DELETE
  path exists, so even a fully-trusted `DOCKER_HOST` env var pointed at this
  proxy cannot restart, create, or exec into anything.
- NAS: bound to loopback (tailnet-reachable only, never the LAN). Box: bound
  to its LAN IP, read-only -- the same LAN posture as dagu/forgejo on the NAS.
- Runs as the official HAProxy image's non-root `haproxy` user; the socket
  is mounted `:ro`; `read_only: true`, `cap_drop: ALL`,
  `no-new-privileges`, no published ports beyond the one bind.

## First deploy

**NAS** (from `/volume1/homes/Rob/mainline/fleet-vendor-stacks`):

```bash
cd /volume1/homes/Rob/mainline/fleet-vendor-stacks/docker-ro-proxy
/var/packages/ContainerManager/target/usr/bin/docker compose --env-file ~/fleet.env up -d --build
```

**robstinybox** (as user `svc`, from `/home/svc/mainline/fleet-vendor-stacks`):

```bash
cd /home/svc/mainline/fleet-vendor-stacks/docker-ro-proxy
docker compose --env-file ~/fleet.env up -d --build
```

After the first manual deploy on each host, the unified deploy engine
(`dagu-dags/scripts/fleet_deploy.py`) keeps it current from the registry
entry.

## Allow/deny verification battery

Run against each host's bound address: from the Mac, `H=100.93.26.74` for
the NAS (tailnet); from a LAN device, `H=192.168.1.201` for robstinybox. Pick a real, currently-running
container `NAME` on that host for the per-container checks.

```bash
H=100.93.26.74   # then repeat with H=192.168.1.201 (from the LAN)
NAME=sonarr      # any running container on that host

# --- ALLOWED (expect 200) ---
curl -s -o /dev/null -w "%{http_code} GET /_ping\n" "http://$H:2375/_ping"
curl -s -o /dev/null -w "%{http_code} GET /version\n" "http://$H:2375/version"
curl -s -o /dev/null -w "%{http_code} GET /containers/json\n" "http://$H:2375/containers/json"
curl -s -o /dev/null -w "%{http_code} GET /containers/$NAME/json\n" "http://$H:2375/containers/$NAME/json"
curl -s -o /dev/null -w "%{http_code} GET /containers/$NAME/logs\n" "http://$H:2375/containers/$NAME/logs?stdout=1&tail=3"

# --- DENIED (expect 403) ---
curl -s -o /dev/null -w "%{http_code} POST /containers/$NAME/stop\n" -X POST "http://$H:2375/containers/$NAME/stop"
curl -s -o /dev/null -w "%{http_code} POST /containers/$NAME/restart\n" -X POST "http://$H:2375/containers/$NAME/restart"
curl -s -o /dev/null -w "%{http_code} POST /containers/create\n" -X POST "http://$H:2375/containers/create"
curl -s -o /dev/null -w "%{http_code} POST /containers/$NAME/exec\n" -X POST "http://$H:2375/containers/$NAME/exec"
curl -s -o /dev/null -w "%{http_code} GET /containers/$NAME/archive\n" "http://$H:2375/containers/$NAME/archive?path=/etc"
curl -s -o /dev/null -w "%{http_code} GET /images/json\n" "http://$H:2375/images/json"
curl -s -o /dev/null -w "%{http_code} DELETE /containers/$NAME\n" -X DELETE "http://$H:2375/containers/$NAME"
```

`GET /_ping` should print `OK` in the body if you drop the `-o /dev/null -w`
and just `curl -s "http://$H:2375/_ping"`.

Docker-CLI form of the same allow/deny pair:

```bash
docker -H tcp://$H:2375 ps                 # works
docker -H tcp://$H:2375 restart "$NAME"    # denied (403 from the proxy)
```

### Binding checks

The NAS proxy must **not** be reachable on its LAN IP (the box's is LAN-bound
by design, see above):

```bash
curl -s -m 3 -o /dev/null -w "%{http_code}\n" http://192.168.1.200:2375/_ping || echo "refused (expected)"
```

On the NAS itself, loopback answers -- this is expected, it's how userspace
`tailscaled` delivers inbound tailnet traffic (see the quirk above):

```bash
ssh nas curl -s http://127.0.0.1:2375/_ping
```
