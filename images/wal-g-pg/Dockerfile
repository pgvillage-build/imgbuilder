# wal-g-pg: wal-g for PostgreSQL, built from source with the same recipe as the wal-g-pg packages in aptbuilder and
# rpmbuilder. libsodium, lzo and brotli are linked statically, so the final image only needs glibc and ca-certificates.

# renovate: datasource=github-releases depName=pgvillage-tools/wal-g versioning=loose
ARG WALG_VERSION=v3.0.9b

# same Debian release as the final image, so the binary links against the same glibc
FROM docker.io/library/debian:trixie-slim@sha256:62bd423aeee912639ca8ee681c2b762cb3e25137e0a92b20be455898a9f034ad AS build

ARG WALG_VERSION
# renovate: datasource=golang-version depName=go
ARG GO_VERSION=1.27.1
ARG TARGETARCH
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update -qq && \
    DEBIAN_FRONTEND=noninteractive apt-get install -qq -y --no-install-recommends \
      ca-certificates curl git cmake build-essential pkg-config liblzo2-dev libsodium-dev && \
    rm -rf /var/lib/apt/lists/*

RUN curl -fsSL --retry 3 "https://go.dev/dl/go${GO_VERSION}.linux-${TARGETARCH}.tar.gz" | tar -xz -C /usr/local
ENV PATH=/usr/local/go/bin:$PATH

WORKDIR /src
RUN git clone -q --depth 1 --branch "${WALG_VERSION}" https://github.com/pgvillage-tools/wal-g.git . && \
    USE_BROTLI=yes USE_LIBSODIUM=yes USE_LZO=yes make deps && \
    go build -tags "brotli libsodium liblzo" -o /wal-g-pg -ldflags "-s -w \
      -X github.com/wal-g/wal-g/cmd/pg.buildDate=$(date -u +%Y.%m.%d_%H:%M:%S) \
      -X github.com/wal-g/wal-g/cmd/pg.gitRevision=$(git rev-parse HEAD) \
      -X github.com/wal-g/wal-g/cmd/pg.walgVersion=${WALG_VERSION}" ./main/pg

# glibc and ca-certificates, no shell; runs as nonroot (UID 65532)
FROM gcr.io/distroless/base-debian13:nonroot@sha256:f9518b74815721f537a7458cc693b4406208c04fce23bbffab511516117f34fd

ARG WALG_VERSION
LABEL org.opencontainers.image.source="https://github.com/pgvillage-build/imgbuilder" \
      org.opencontainers.image.description="wal-g for PostgreSQL (wal-g-pg), built from pgvillage-tools/wal-g" \
      org.opencontainers.image.version="${WALG_VERSION}" \
      org.opencontainers.image.licenses="Apache-2.0"

COPY --from=build /wal-g-pg /usr/local/bin/wal-g-pg
ENTRYPOINT ["/usr/local/bin/wal-g-pg"]
