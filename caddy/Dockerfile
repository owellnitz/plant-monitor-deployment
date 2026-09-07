# Caddy is not shipped with DNS provider modules, so it has to be rebuilt with
# the deSEC one. We need the DNS-01 ACME challenge (not HTTP-01) because the
# host is LAN-only: Let's Encrypt never has to reach it, it only checks a TXT
# record that Caddy sets via the deSEC API.
FROM caddy:2.11.4-builder AS builder

RUN xcaddy build --with github.com/caddy-dns/desec

FROM caddy:2.11.4

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
