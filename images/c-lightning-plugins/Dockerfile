# Core Lightning publishes release binaries for amd64 only, so this image is
# linux/amd64 only. Build with: docker buildx build --platform linux/amd64 .

ARG UBUNTU_VERSION=24.04
# Tag of boltz/bitcoin-core to take bitcoin-cli from (bcli plugin shells out to it)
ARG BITCOIN_VERSION=latest

FROM --platform=linux/amd64 boltz/bitcoin-core:${BITCOIN_VERSION} AS bitcoin-core

FROM --platform=linux/amd64 ubuntu:${UBUNTU_VERSION} AS builder
ARG UBUNTU_VERSION
ARG CLN_VERSION=26.06.9
# sha256 of clightning-v${CLN_VERSION}-Ubuntu-${UBUNTU_VERSION}-amd64.tar.xz, from
# SHA256SUMS-v${CLN_VERSION}. Update this whenever CLN_VERSION or UBUNTU_VERSION changes.
ARG CLN_SHA256=2f5b55838f8de8a7a627f8e34f888afb6770c5f303deee799edac8595bc3d2d1
ARG HOLD_VERSION=v0.3.3
ARG CLNURL_VERSION=v0.1.0

RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates curl xz-utils && \
    rm -rf /var/lib/apt/lists/*

# Core Lightning release tarball. Unpacks to ./usr/bin, ./usr/libexec/c-lightning,
# ./usr/share -- lightningd finds its subdaemons/plugins at ../libexec relative to
# its own path, so this layout must be preserved in the final image.
WORKDIR /cln
RUN curl -fsSL -o cln.tar.xz \
      "https://github.com/ElementsProject/lightning/releases/download/v${CLN_VERSION}/clightning-v${CLN_VERSION}-Ubuntu-${UBUNTU_VERSION}-amd64.tar.xz" && \
    echo "${CLN_SHA256}  cln.tar.xz" | sha256sum -c - && \
    mkdir -p rootfs && tar -xf cln.tar.xz -C rootfs && rm cln.tar.xz

# Download hold plugin
WORKDIR /hold
RUN curl -fsSL https://github.com/BoltzExchange/hold/releases/download/${HOLD_VERSION}/hold-linux-amd64.tar.gz -o hold.tar.gz
RUN tar -xvf hold.tar.gz && mv build/hold-linux-amd64 hold

# Download clnurl plugin
WORKDIR /clnurl
RUN curl -fsSL https://github.com/michael1011/clnurl/releases/download/${CLNURL_VERSION}/clnurl.tar.gz -o clnurl.tar.gz
RUN tar -xvf clnurl.tar.gz

FROM --platform=linux/amd64 ubuntu:${UBUNTU_VERSION}

# Runtime deps of the release binaries: libsodium.so.23, libsqlite3.so.0, libpq.so.5.
# libevent-dev + sqlite3 kept for parity with the boltz/c-lightning image.
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      libpq5 \
      libsodium23 \
      libsqlite3-0 \
      libevent-dev \
      sqlite3 && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /cln/rootfs/usr/bin/lightningd /usr/bin/lightningd
COPY --from=builder /cln/rootfs/usr/bin/lightning-cli /usr/bin/lightning-cli
COPY --from=builder /cln/rootfs/usr/bin/lightning-hsmtool /usr/bin/lightning-hsmtool
COPY --from=builder /cln/rootfs/usr/libexec/c-lightning /usr/libexec/c-lightning

COPY --from=bitcoin-core /bin/bitcoin-cli /bin/bitcoin-cli

COPY --from=builder /hold/hold /usr/local/bin/hold
COPY --from=builder /clnurl/clnurl /usr/local/bin/clnurl

ENTRYPOINT ["lightningd"]
