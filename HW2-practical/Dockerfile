# Caddy has no built-in JWT support, so we build a custom binary that
# includes the caddy-jwt module (https://github.com/ggicci/caddy-jwt).
FROM docker.arvancloud.ir/caddy:2-builder AS builder
ENV GOPROXY="https://goproxy.cn,direct"
RUN xcaddy build --with github.com/ggicci/caddy-jwt

FROM docker.arvancloud.ir/caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
COPY Caddyfile /etc/caddy/Caddyfile
