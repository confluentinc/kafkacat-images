#
# Copyright 2019 Confluent Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG DOCKER_UPSTREAM_REGISTRY
ARG DOCKER_UPSTREAM_TAG=latest
ARG PROJECT_VERSION
ARG ARTIFACT_ID

FROM ${DOCKER_UPSTREAM_REGISTRY}confluentinc/cp-base-new:${DOCKER_UPSTREAM_TAG}

WORKDIR /build

ENV VERSION=1.3.1-1
ENV BUILD_PACKAGES="build-essential cmake python git curl zlib1g-dev libsasl2-dev libssl-dev"

RUN echo "Building kafkacat ....." \
    && yum -q -y update \
    && yum -q install -y $BUILD_PACKAGES \
    && yum clean all \
    && git clone https://github.com/edenhill/kafkacat \
    && cd kafkacat \
    && git checkout tags/debian/$VERSION \
    && ./bootstrap.sh \
    && make

FROM ${DOCKER_UPSTREAM_REGISTRY}confluentinc/cp-base-new:${DOCKER_UPSTREAM_TAG}

# Make sure you use an appropriate contact address.
LABEL maintainer="partner-support@confluent.io"
LABEL io.confluent.docker=true
ARG COMMIT_ID=unknown
LABEL io.confluent.docker.git.id=$COMMIT_ID
ARG BUILD_NUMBER=-1
LABEL io.confluent.docker.build.number=$BUILD_NUMBER

COPY --from=0 /build/kafkacat/kafkacat /usr/local/bin/

RUN yum -q -y update \
    && echo "Installing runtime dependencies for SSL and SASL support ...." \
    && yum -q -y update \
    && yum install -y \
        openssl \
        libssl1.1 \
        libsasl2-2 \
        libsasl2-modules-gssapi-mit \
        krb5-user \
        krb5-config \
        ca-certificates \
    && echo "===> clean up ..."  \
    && yum clean all \
    && rm -rf /tmp/*

CMD ["kafkacat"]
