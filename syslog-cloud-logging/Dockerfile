FROM debian:10.10-slim

LABEL maintainer="pedrorouremalta@gmail.com"

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
      curl \
      gnupg2 \
      ca-certificates \
      lsb-release \
      apt-utils

RUN curl -sSO https://dl.google.com/cloudagents/add-logging-agent-repo.sh \
    && chmod +x add-logging-agent-repo.sh \
    && ./add-logging-agent-repo.sh --also-install

COPY nginx-app-protect.conf /etc/google-fluentd/config.d/

CMD ["/usr/sbin/google-fluentd"]