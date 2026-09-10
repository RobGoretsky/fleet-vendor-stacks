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
