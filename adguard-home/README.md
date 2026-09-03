# adguard-home

Fleet DNS server (AdGuard Home), deployed on the **NAS** by the unified
engine off this repo's `compose.yaml`.

**Why host network**: bridge mode's docker-proxy has a UDP hairpin bug
that stops other containers on the same host from resolving fleet DNS
names (`.rig`/`.nas`/`.box`) through it. Host network removes that hop --
standard for a DNS server, and AdGuard is already reached via the host's
own IP either way, so nothing about how it's *used* changes.

**The `http.address` change**: in bridge mode the web UI listened on `:80`
internally, published out as `:8088`. In host mode there's no port
mapping to do that translation, so `AdGuardHome.yaml`'s `http.address`
must be edited to `0.0.0.0:8088` directly, or `adguard.nas:8088` (the
registry's `dns:` entry) breaks. This edit happens once, at cutover, via
`docker cp` (procedure in the `dns-manager` skill) -- it isn't part of
this compose because it's a one-time config migration, not stack
behavior.

Full cutover steps (stop, edit, `docker rm -f`, merge, verify UDP
resolution): Obsidian `Tech-Learning/Wave3-HA-Cutover` §B.
