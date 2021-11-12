FROM registry as builder

FROM alpine:3.14.2

RUN set -ex \
    && apk --no-cache upgrade \
    && apk add --no-cache ca-certificates apache2-utils

COPY --from=builder /bin/registry /bin/registry
COPY --from=builder /etc/docker/registry/config.yml /etc/docker/registry/config.yml
COPY --from=builder /entrypoint.sh /entrypoint.sh

LABEL source=git@github.com:kyma-project/kyma.git

VOLUME ["/var/lib/registry"]
EXPOSE 5000

ENTRYPOINT ["/entrypoint.sh"]
CMD ["/etc/docker/registry/config.yml"]
