# built with `docker build . -t ghcr.io/renklus/paperless-ngx-backup:v9`
# published with `docker push ghcr.io/renklus/paperless-ngx-backup:v9`

FROM ghcr.io/paperless-ngx/paperless-ngx:2.20.3
LABEL org.opencontainers.image.source=https://github.com/renklus/home-network

RUN rm -rf /etc/s6-overlay/s6-rc.d/init-migrations \
    && rm -f /etc/s6-overlay/s6-rc.d/init-complete/dependencies.d/init-migrations

VOLUME ["/usr/src/paperless/data", \
        "/usr/src/paperless/media", \
        "/usr/src/paperless/export"]

ENTRYPOINT ["/init"]