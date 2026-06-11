FROM debian:bookworm-slim

USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    dpkg \
    gzip \
    tar \
  && rm -rf /var/lib/apt/lists/*

RUN groupadd -g 1000660000 sandbox && \
    useradd -u 1000660000 -g sandbox -m -d /sandbox -s /bin/bash sandbox && \
    mkdir -p \
      /sandbox/bin \
      /sandbox/.local/bin \
      /sandbox/.cache \
      /sandbox/tmp \
      /sandbox/work && \
    chown -R sandbox:sandbox /sandbox

USER sandbox
ENV HOME=/sandbox
ENV PATH=/sandbox/bin:/sandbox/.local/bin:/usr/local/bin:/usr/bin:/bin
ENV TMPDIR=/sandbox/tmp
WORKDIR /sandbox
