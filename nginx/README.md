# nginx

robstinybox's fleet-wide reverse proxy. Replaces the box's native nginx
(`Tech-Learning/Rig-DNS-Staging-Pack` §C) and the staged-sudo-paste
workflow that came with it -- ruling 4 in
`Tech-Learning/Fleet-Centralization-Phase2`.

**Config is split in two, only one half lives here:**

- `nginx.conf` (this dir, checked in): the static http{} block, the
  websocket-upgrade `map`, and `include /etc/nginx/conf.d/*.conf`. Edited
  by hand, like any other checked-in config.
- `conf.d/*.conf` (NOT in this repo -- `${APPDATA_ROOT}/nginx/conf.d` on
  robstinybox, bind-mounted read-only): rendered by `fleet_deploy.py` on
  every run, from every registry entry's `dns:` field (one server block
  per `name:port` pair, for entries whose `host:` includes robstinybox --
  `server_name` expanded to both the `.rig` and `.nas` retained-alias
  names, `Rig-DNS-Staging-Pack` §F, even though the registry only states
  one canonical name) plus a few hand-maintained static blocks it also
  owns outright -- `dagu.box`, the `.family` names, and the
  default-server catch-all (§G/§H). The render is idempotent (a `.conf`
  file is only rewritten when its content changed) and the engine sends
  `docker kill -s HUP nginx` after any change, so a new registry `dns:`
  entry goes live on the next `fleet-deploy` tick with no PR to this repo
  and no sudo.

**The retained `.nas` alias is kept, not dropped**: every mover's
`server_name` still carries both `foo.rig` and `foo.nas` -- the render
derives the pair automatically from whichever one the registry's `dns:`
field states, rather than needing both written out by hand. This is
load-bearing, not cosmetic: `forgejo.nas` is embedded in every gated git
remote and in `registry.yaml`'s own `repo:` fields until the
`forgejo.rig` rename lands, and `dominion.nas` / `fleet.nas` are live
registry `url:` values elsewhere.

**Cutover**: this container binds host port 80, same as the native nginx
it replaces, so it fails to bind and crash-loops (harmless,
`restart: unless-stopped`) until Rob's one `sudo systemctl disable --now
nginx` after this PR (and the registry PR wiring it into `deploy:`) land.
See the compose file's header and the PR body for the ordering.

## TLS

`Home/Fleet-HTTPS` fronts every fleet name with HTTPS via a private CA --
one leaf, ~22+ SANs, re-signed on the laptop whenever a name is added.
`nginx.conf` sets `ssl_certificate`/`ssl_certificate_key` **once**, at the
`http{}` level, so every rendered block in `conf.d/` inherits it with no
per-block cert config -- `fleet_deploy.py`'s `_nginx_server_block()` just
adds `listen 443 ssl;` next to `listen 80;` on every block (dynamic and
static alike), plus the catch-all.

The two files it points at --
`/etc/nginx/fleet-tls/fleet.crt`/`fleet.key` inside the container, bind-mounted
read-only from `${APPDATA_ROOT}/nginx/tls` on the host -- are **required,
not optional**: nginx refuses to start at all if either is missing or
unreadable. There is deliberately no port-80-only fallback (matching
`Home/Fleet-HTTPS`'s design elsewhere), so landing those two files on the
host is a precondition of this container coming up, not a nice-to-have.
See the PR body for the one-time copy (this replaces the box's old
standalone `tls.conf`, which the original nginx-container cutover had
dropped entirely -- nothing served `:443` on the box from that point until
this PR).

## Ad-hoc names (ruling 11)

Any inner-loop container listening on `127.0.0.1:<port>` on robstinybox is
reachable as `<anything>--<port>.rig` with **no registry entry and no PR** --
`fleet_deploy.py` renders one extra static block, `_wildcard_rig.conf`, whose
`server_name` is a regex (`~^(?<app>[a-z0-9-]+)--(?<port>\d{4,5})\.rig$`)
that captures the port straight out of the hostname and proxies to it.
nginx matches `server_name` exact names first, then wildcards, then regex
last, so any real registered `dns:` name's own exact block always wins --
this one only ever answers for a name nothing else claims. It depends on
`Tech-Learning/Rig-DNS-Staging-Pack` §J's `*.rig` → box AdGuard wildcard
rewrite to route the request here at all; a registered prototype instead
gets a real name via the registry's `dns:` field (`robapp graduate` /
`registry.yaml`), same as any other mover.

**HTTP-only by design, permanently.** The `_wildcard_rig.conf` block still
carries `listen 443 ssl;` for a uniform render, but it can never present a
*valid* cert: the TLS leaf above enumerates its SANs explicitly from the
registry, and an ad-hoc name has no registry entry to be enumerated from.
So `https://<name>--<port>.rig` will complete a TLS handshake but fail
hostname verification -- there is no path to a real cert for one of these
names short of graduating it to a registered `dns:` entry and waiting for
the next `fleet_ca.py --renew`. Treat `<name>--<port>.rig` as `http://`
only.
