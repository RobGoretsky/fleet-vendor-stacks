# forgejo

The fleet's git gate: origin for every gated repo, branch protection, the
`claude-bot` account, the push-mirrors to the GitHub twins, and the two
Forgejo **system webhooks** that fire `fleet-deploy` on each host. Deployed
on **robstinybox** by the unified engine off this dir's `compose.yaml` --
read that file's header for the per-setting rationale (why `user:` and not
`USER_UID`, where `app.ini` lives, why push mirrors need no config here).

This is a one-time **migration from the NAS**, not a fresh install. The NAS
ran it as a hand-installed appliance from
`/volume1/docker/forgejo/docker-compose.yml`, which was never git-tracked;
this stack is that compose, pinned, path-varibalized, and brought under the
PR gate. Full cutover runbook: Obsidian
`Tech-Learning/Wave5-Forgejo-Cutover`. This file covers only the data-dir
copy, which is the whole of the bring-up.

## The data-dir copy

**Everything that matters is in the data dir.** The GitHub twins mirror repo
*content* and nothing else -- users, access tokens, branch protection rules,
the two system webhooks and their `dagu_wh_` tokens, and the push-mirror
rows (plus the mirror PAT, which lives in a `push_mirror_<id>` git remote
inside each bare repo) exist only here. `app.ini` -- and with it
`SECRET_KEY`, `INTERNAL_TOKEN` and the JWT secrets that make those tokens
decryptable -- is at `custom/conf/app.ini` inside this same tree, not in a
separate `/etc/gitea` volume. So: copy the dir and everything comes with it;
reinstall fresh and both webhooks and both `dagu_wh_` tokens have to be
recreated (they are display-once -- recovery is regeneration).

### Pre-flight, on the NAS

The container runs as uid 1000 and `docker-setup.sh` chmods `custom/` and
`git/` to `0700`, so **`Rob` (uid 1026) cannot read them** -- a plain
`tar` as Rob silently skips the most important subtrees. Check first, and
note the size while you are there:

```
ssh -p 7600 Rob@192.168.1.200 'ls -ln /volume1/docker/forgejo/data && du -sh /volume1/docker/forgejo/data'
```

### The copy itself

Run **on the box**. `rsync -e ssh` to a Synology fails unless DSM's rsync
service is enabled (wave 3 hit this), so it is tar over ssh -- and the tar
runs inside a throwaway container **as uid 1000** so it can actually read
the `0700` dirs (the same trick wave 4 used for the `claude-home` volume):

```
sudo -u svc mkdir -p /home/svc/appdata/forgejo/data
ssh -n -p 7600 Rob@192.168.1.200 \
  '/var/packages/ContainerManager/target/usr/bin/docker run --rm -u 1000:1000 \
     -v /volume1/docker/forgejo/data:/d alpine:3.20 tar czf - -C /d .' \
  | sudo -u svc tar xzf - -C /home/svc/appdata/forgejo/data
sudo chown -R svc:svc /home/svc/appdata/forgejo/data
```

Extracting as `svc` already makes everything svc-owned (a non-root `tar`
cannot chown), so the `chown` is belt-and-braces -- run it anyway, since
`user: "3000:3000"` in the compose has no fallback if one file is wrong.

**If the engine has already started the stack against an empty dir**
(expected -- the registry PR merges *before* the copy, so the box comes up
on a fresh install), stop it and clear the generated tree before extracting,
or you get a hybrid of the NAS's `app.ini` and a freshly generated one:

```
sudo -u svc docker compose --env-file /home/svc/fleet.env -f \
  /home/svc/mainline/fleet-vendor-stacks/forgejo/compose.yaml stop forgejo
sudo rm -rf /home/svc/appdata/forgejo/data && sudo -u svc mkdir -p /home/svc/appdata/forgejo/data
```

## Verify after start

- `curl -s http://192.168.1.201:3001/api/v1/version` -- the same version the
  NAS reported (`16.0.3+gitea-1.22.0` at prep time).
- Repo list matches (12 repos at prep time), and a clone works over HTTP.
- Git hooks: `docker exec forgejo /usr/local/bin/gitea admin regenerate hooks
  -c /var/lib/gitea/custom/conf/app.ini` -- idempotent, and the cheapest way
  to be sure the per-repo hook scripts point at this container's binary.
- Site Admin -> Webhooks shows **both** system hooks with their tokens
  intact; Site Admin -> Users shows `robg` and `claude-bot`.
- Each repo's Settings -> Mirror Settings still lists its GitHub twin.

## Rollback

`docker start forgejo` on the NAS (stopped, never removed, at cutover) and
flip the AdGuard rewrites back to the NAS. The NAS's
`/volume1/docker/forgejo/` stays untouched until the day-75 verdict. Full
steps: runbook §E.
