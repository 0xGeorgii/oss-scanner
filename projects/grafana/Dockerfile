# Build the OSS Grafana server and frontend while network access is available.
# The scanner copies this Dockerfile into the Grafana checkout before building.
FROM golang:1.27.2-bookworm AS go-toolchain

FROM node:24-bookworm
COPY --from=go-toolchain /usr/local/go /usr/local/go
ENV PATH="/usr/local/go/bin:${PATH}" \
    NODE_OPTIONS="--max-old-space-size=8192"

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential git make python3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
COPY . .

# Keep both dependency trees in the final image for tests without Internet access.
RUN yarn install --immutable
RUN go mod download

# Follow the build commands in Grafana's own Dockerfile, using the OSS tags.
RUN yarn build
RUN make build-go GO_BUILD_TAGS=oss WIRE_TAGS=oss
