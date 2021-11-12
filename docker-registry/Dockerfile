FROM registry@sha256:c11a277a91045f91866550314a988f937366bc2743859aa0f6ec8ef57b0458ce as builder

FROM alpine:3.14.2

RUN set -ex \
    && apk --no-cache upgrade \
    && apk add --no-cache ca-certificates apache2-utils

COPY --from=builder /bin/registry /bin/registry
COPY --from=builder /etc/docker/registry/config.yml /etc/docker/registry/config.yml
COPY --from=builder /entrypoint.sh /entrypoint.sh

VOLUME ["/var/lib/registry"]
EXPOSE 5000

LABEL source=git@github.com:kyma-project/kyma.git

ENTRYPOINT ["/entrypoint.sh"]
CMD ["/etc/docker/registry/config.yml"]
