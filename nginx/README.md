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
  per `name:port` pair, for entries whose `host:` includes robstinybox)
  plus a few hand-maintained static blocks it also owns outright --
  `dagu.box`, the `.family` names, and the default-server catch-all
  (`Tech-Learning/Rig-DNS-Staging-Pack` §G/§H). The render is idempotent
  (a `.conf` file is only rewritten when its content changed) and the
  engine sends `docker kill -s HUP nginx` after any change, so a new
  registry `dns:` entry goes live on the next `fleet-deploy` tick with no
  PR to this repo and no sudo.

**Known gap vs. the old hand-written config**: the retained `.nas` alias
(every mover's old §C block listed both `foo.rig` and `foo.nas` as
`server_name`) is gone -- the registry's `dns:` field carries one
canonical name per port, and the render only emits what's there. This is
an intentional simplification under ruling 4 ("no new name needs a
paste"), not an oversight; flagged in the PR for Rob to confirm.

**Cutover**: this container binds host port 80, same as the native nginx
it replaces, so it fails to bind and crash-loops (harmless,
`restart: unless-stopped`) until Rob's one `sudo systemctl disable --now
nginx` after this PR (and the registry PR wiring it into `deploy:`) land.
See the compose file's header and the PR body for the ordering.
