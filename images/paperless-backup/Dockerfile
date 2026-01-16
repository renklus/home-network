FROM ghcr.io/paperless-ngx/paperless-ngx:2.20.3

CMD ["document_exporter"]

VOLUME ["/usr/src/paperless/data", \
        "/usr/src/paperless/media", \
        "/usr/src/paperless/export"]

ENTRYPOINT ["/init"]