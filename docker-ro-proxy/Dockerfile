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

COPY haproxy-docker-ro.cfg /usr/local/etc/haproxy/haproxy.cfg
