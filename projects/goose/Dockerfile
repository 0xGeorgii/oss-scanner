FROM node:24-bookworm AS node
FROM rust:bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    GOOSE_DISABLE_KEYRING=1 \
    CARGO_INCREMENTAL=0 \
    RUST_MIN_STACK=8388608

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential ca-certificates clang cmake curl git pkg-config \
      libssl-dev libdbus-1-dev libxcb1-dev libclang-dev \
      protobuf-compiler libprotobuf-dev python3 python3-venv \
      libgtk-3-0 libnss3 libasound2 libgbm1 xvfb \
    && rm -rf /var/lib/apt/lists/*

COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/npm
RUN ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
    && npm install --global pnpm@10.30.3

COPY . /src
WORKDIR /src

RUN cargo fetch --locked \
    && cargo build --locked --workspace \
    && cargo test --locked --workspace --no-run

RUN rm -f ui/desktop/src/bin/goose \
    && cp target/debug/goose ui/desktop/src/bin/goose \
    && pnpm --dir ui install --frozen-lockfile \
    && pnpm --dir ui/desktop run package

CMD ["bash"]
