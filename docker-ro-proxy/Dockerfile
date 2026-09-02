# Read-only docker proxy for the fleet cockpit -- official HAProxy image +
# our allowlist config (haproxy-docker-ro.cfg, which IS the security policy
# -- read its header). Same rationale as claude-remote-host's control proxy
# for not using tecnativa/docker-socket-proxy: its env-flag rules are
# path-scoped, not method-scoped, and this proxy wants an explicit GET-only
# allowlist, not a flag combination.
#
# Pinned by digest; the official image runs as the non-root `haproxy` user --
# compose adds the host docker group (DOCKER_GID, per-host, from
# ~/fleet.env) so it can read the read-only-mounted socket.
FROM haproxy:3.2.21-alpine@sha256:66e25cc9a8332635f4e897f7f4b1e5622c25f09f0ee23cddc6ce9bdb3a24772a

# --chmod: COPY otherwise keeps the checkout file's mode, and a checkout made
# under umask 077 (the NAS's default) yields a 600 root-owned cfg the non-root
# haproxy user cannot open (crash loop, seen on the NAS first deploy
# 2026-09-01). A separate RUN chmod does NOT work: the base image already
# switched to USER haproxy, so chmod on a root-owned file fails (broke
# fleet-deploy on both hosts 2026-09-01). --chmod applies at copy time as
# root; BuildKit (the default builder on both hosts) supports it.
COPY --chmod=644 haproxy-docker-ro.cfg /usr/local/etc/haproxy/haproxy.cfg
