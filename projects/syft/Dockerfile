FROM golang:1.26-bookworm
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl git make skopeo zip unzip \
    && rm -rf /var/lib/apt/lists/*

# oras pulls the prebuilt test fixture cache (version pinned in .binny.yaml)
ARG ORAS_VERSION=1.3.4
RUN curl -fsSL "https://github.com/oras-project/oras/releases/download/v${ORAS_VERSION}/oras_${ORAS_VERSION}_linux_$(go env GOARCH).tar.gz" \
    | tar -xz -C /usr/local/bin oras

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# build tags must match .make/main.go / .goreleaser.yaml. The skips cover tests this environment can never pass:
#   Test_errors expects a permission error, which root (the scanner's user) never gets
#   Test_EnvironmentTask needs docker to build its image fixtures (they aren't in the oras cache since the dirs
#   under internal/os/testdata aren't named image-*, which is how fingerprint_docker_fixtures.py finds them)
ENV GOFLAGS="-tags=containers_image_openpgp,exclude_graphdriver_btrfs -skip=^(Test_errors|Test_EnvironmentTask)$"

RUN go mod download
RUN go build -o /usr/local/bin/syft ./cmd/syft
ENV SYFT_CHECK_FOR_APP_UPDATE=false

# unit tests depend on generated fixtures (testdata/Makefile outputs and stereoscope image tars). `make unit` pulls these
# from ghcr with oras and needs docker to check freshness, neither of which works offline, so pull them here instead.
RUN oras pull ghcr.io/anchore/oss-cache/syft-test-fixture-cache:v2

# image references like `alpine:latest` need a daemon or registry, neither of which exists offline, so save a few
# images as archives to scan instead: syft docker-archive:/images/alpine.tar
RUN mkdir /images \
    && skopeo copy docker://alpine:latest docker-archive:/images/alpine.tar:alpine:latest \
    && skopeo copy docker://debian:bookworm-slim docker-archive:/images/debian.tar:debian:bookworm-slim

# run the unit tests (everything outside test/ dirs, same as `make unit`) so the build and test caches are warm offline
RUN go test $(go list ./... | grep -v /test/) || echo "WARNING: unit tests reported failures"
