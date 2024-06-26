# Copyright 2018 Google, Inc. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# image builder base on golang:1.21.5-alpine3.19
FROM europe-docker.pkg.dev/kyma-project/prod/external/golang@sha256:55f716237933c85cee01748700755b4ac8736fb1ca974c9aed051691b68d6dc2 as builder


ARG KANIKO_VERSION
ARG GOARCH=amd64
WORKDIR /go/src/github.com/GoogleContainerTools/kaniko

RUN echo $GOARCH > /goarch

#This arg is passed by docker buildx and contains the platform info in the form linux/amd64, linux/ppc64le etc.
ARG TARGETPLATFORM

#Capture ARCH has write to /goarch
RUN [ ! "x" = "x$TARGETPLATFORM" ] && `echo $TARGETPLATFORM |  awk '{split($0,a,"/"); print a[2]}' > /goarch` || echo "$GOARCH"

RUN apk upgrade && apk add git make build-base bash
# Get GCR credential helper
RUN GOARCH=$(cat /goarch) && CGO_ENABLED=0 && \
  (mkdir -p /go/src/github.com/GoogleCloudPlatform || true)                  && \
  cd /go/src/github.com/GoogleCloudPlatform                                  && \
  git clone https://github.com/GoogleCloudPlatform/docker-credential-gcr.git && \
  cd /go/src/github.com/GoogleCloudPlatform/docker-credential-gcr            && \
  go build -ldflags "-linkmode external -extldflags -static" -o /usr/local/bin/docker-credential-gcr main.go

# Get Amazon ECR credential helper
RUN GOARCH=$(cat /goarch) && GO111MODULE=off go get -u github.com/awslabs/amazon-ecr-credential-helper/ecr-login/cli/docker-credential-ecr-login && \
make -C /go/src/github.com/awslabs/amazon-ecr-credential-helper

# ACR docker env credential helper
RUN GOARCH=$(cat /goarch) && (mkdir -p /go/src/github.com/chrismellard || true)   && \
  cd /go/src/github.com/chrismellard && \
  git clone https://github.com/chrismellard/docker-credential-acr-env && \
  cd docker-credential-acr-env && \
  make build

# Add .docker config dir
RUN mkdir -p /kaniko/.docker

RUN git clone --depth 1 --branch v$KANIKO_VERSION https://github.com/GoogleContainerTools/kaniko . \
  && go mod edit -replace code.gitea.io/sdk/gitea=code.gitea.io/sdk/gitea@v0.15.1 \
  && go mod vendor \
  && make GOARCH=$(cat /goarch)

# Generate ca-certificates from debian:buster-slim
FROM debian@sha256:5b0b1a9a54651bbe9d4d3ee96bbda2b2a1da3d2fa198ddebbced46dfdca7f216 AS certs

RUN \
  apt update && \
  apt install -y ca-certificates && \
  cat /etc/ssl/certs/* > /ca-certificates.crt

FROM scratch
COPY --from=0 /go/src/github.com/GoogleContainerTools/kaniko/out/executor /workspace/kaniko/executor
COPY --from=0 /usr/local/bin/docker-credential-gcr /workspace/kaniko/docker-credential-gcr
COPY --from=0 /go/src/github.com/awslabs/amazon-ecr-credential-helper/bin/local/docker-credential-ecr-login /workspace/kaniko/docker-credential-ecr-login
COPY --from=0 /go/src/github.com/chrismellard/docker-credential-acr-env/build/docker-credential-acr-env /workspace/kaniko/docker-credential-acr
COPY --from=certs /ca-certificates.crt /workspace/kaniko/ssl/certs/
COPY --from=0 /kaniko/.docker /workspace/kaniko/.docker
COPY --from=0 /go/src/github.com/GoogleContainerTools/kaniko/files/nsswitch.conf /etc/nsswitch.conf
ENV HOME /root
ENV USER root
ENV PATH /usr/local/bin:/workspace/kaniko
ENV SSL_CERT_DIR=/kaniko/ssl/certs
ENV DOCKER_CONFIG /workspace/kaniko/.docker/
ENV DOCKER_CREDENTIAL_GCR_CONFIG /workspace/kaniko/.config/gcloud/docker_credential_gcr_config.json
WORKDIR /workspace
RUN ["docker-credential-gcr", "config", "--token-source=env"]

ENTRYPOINT ["/workspace/kaniko/executor"]
