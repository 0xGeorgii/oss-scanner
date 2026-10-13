# syntax=docker/dockerfile:1.19

FROM node:24-bookworm AS node
FROM rust:bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    GOOSE_DISABLE_KEYRING=1 \
    CARGO_BUILD_JOBS=2 \
    CARGO_INCREMENTAL=0 \
    CARGO_PROFILE_DEV_DEBUG=1 \
    CARGO_PROFILE_TEST_DEBUG=1 \
    CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_RUSTFLAGS="-C target-feature=+fp16 -C link-arg=-fuse-ld=lld" \
    CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUSTFLAGS="-C link-arg=-fuse-ld=lld" \
    RUST_MIN_STACK=8388608

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential ca-certificates clang cmake curl git lld pkg-config \
      libssl-dev libdbus-1-dev libxcb1-dev libclang-dev \
      protobuf-compiler libprotobuf-dev python3 python3-venv \
      libgtk-3-0 libnss3 libasound2 libgbm1 xvfb \
    && rm -rf /var/lib/apt/lists/*

COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/npm
RUN ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
    && npm install --global pnpm@10.30.3

COPY --exclude=.git --exclude=.oss-scanner/Dockerfile . /src
WORKDIR /src

RUN cargo fetch --locked
RUN cargo build --locked --workspace
RUN cargo test --locked --workspace --no-run \
      --test git_command_security \
      --test schedule_tool_security \
      --test tool_inspection_permission_precedence \
      --test acp_transport_auth_test

RUN rm -f ui/desktop/src/bin/goose \
    && cp target/debug/goose ui/desktop/src/bin/goose \
    && pnpm --dir ui install --frozen-lockfile \
    && pnpm --dir ui/desktop run package

CMD ["bash"]
