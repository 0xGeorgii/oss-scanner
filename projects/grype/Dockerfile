FROM golang:1.26-bookworm
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl git make skopeo zstd \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# build tags must match .make/main.go / .goreleaser.yaml. The skips cover tests this environment can never pass:
#   TestPrepareWritableDB expects a permission error under /root, which root (the scanner's user) never gets
#   TestSyftLocationExcludes needs docker to build its image fixture (grype has no downloadable fixture cache)
ENV GOFLAGS="-tags=containers_image_openpgp,exclude_graphdriver_btrfs -skip=^(TestPrepareWritableDB|TestSyftLocationExcludes)$"

RUN go mod download
RUN go build -o /usr/local/bin/grype ./cmd/grype

# the gosymbols matcher tests build these on demand and fetch modules to do it, so build them while online
RUN make -C grype/matcher/golang/testdata fixtures

# grab a vulnerability DB so the CLI can actually match offline
RUN grype db update || echo "WARNING: grype db update failed"
# don't reach for the network on every run once offline
ENV GRYPE_DB_AUTO_UPDATE=false \
    GRYPE_CHECK_FOR_APP_UPDATE=false

# image references like `alpine:latest` need a daemon or registry, neither of which exists offline, so save a few
# images as archives to scan instead: grype docker-archive:/images/alpine.tar
RUN mkdir /images \
    && skopeo copy docker://alpine:latest docker-archive:/images/alpine.tar:alpine:latest \
    && skopeo copy docker://debian:bookworm-slim docker-archive:/images/debian.tar:debian:bookworm-slim

# run the unit tests (everything outside test/ dirs, same as `make unit`) so the build and test caches are warm offline
RUN go test $(go list ./... | grep -v /test/) || echo "WARNING: unit tests reported failures"
