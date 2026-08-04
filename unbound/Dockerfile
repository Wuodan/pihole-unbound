FROM alpine:3.24

RUN apk add --no-cache \
        ca-certificates \
        curl \
        unbound \
    && mkdir -p /var/lib/unbound \
    && curl --fail --silent --show-error \
        https://www.internic.net/domain/named.root \
        --output /var/lib/unbound/root.hints \
    && chown -R unbound:unbound /var/lib/unbound

COPY unbound.conf /etc/unbound/unbound.conf

EXPOSE 5335/tcp 5335/udp

# unbound-anchor returns 1 after successfully creating/updating the anchor and 0
# when no update is needed (and, unusually, for most errors). Do not chain it
# directly with && or a clean container exits before starting Unbound.
CMD ["sh", "-c", "unbound-anchor -a /var/lib/unbound/root.key; chown unbound:unbound /var/lib/unbound/root.key && exec unbound -d -c /etc/unbound/unbound.conf"]
