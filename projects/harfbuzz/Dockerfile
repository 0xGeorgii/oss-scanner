# Copyright 2016 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
################################################################################

# The scanner supplies a HarfBuzz checkout as the build context. Everything
# needed to rebuild or run tests must be present before the offline audit.
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive \
    CC=clang \
    CXX=clang++ \
    PATH=/opt/venv/bin:$PATH

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential ca-certificates clang libclang-rt-18-dev llvm gdb git \
      pkg-config python3-venv ragel \
      libcairo2-dev libfreetype6-dev libglib2.0-dev libgraphite2-dev \
      libicu-dev libpng-dev zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

COPY . /src
WORKDIR /src

# Use the same Meson, Ninja and FontTools versions as upstream CI. The virtual
# environment and installed packages remain available without network access.
RUN python3 -m venv /opt/venv \
    && pip install --no-cache-dir --require-hashes -r .ci/requirements.txt

# Normal build, with debug information, assertions and frame pointers. Keep
# libraries in the build tree so the tools use this checkout's HarfBuzz.
RUN meson setup build --wrap-mode=nofallback --buildtype=debugoptimized \
      -Db_ndebug=false \
      -Dc_args=-fno-omit-frame-pointer -Dcpp_args=-fno-omit-frame-pointer \
      -Dglib=enabled -Dgobject=enabled -Dfreetype=enabled -Dcairo=enabled \
      -Dicu=enabled -Dgraphite2=enabled -Dpng=enabled -Dzlib=enabled \
      -Dsubset=enabled -Draster=enabled -Dvector=enabled -Dgpu=enabled \
      -Dgpu_demo=disabled -Dtests=enabled -Dutilities=enabled \
      -Ddocs=disabled -Dintrospection=disabled \
    && meson compile -C build -j "$(nproc)"

# A second tree instruments the libraries, command-line tools and standalone
# fuzz drivers, and includes experimental APIs such as dependency analysis.
RUN meson setup build-asan --wrap-mode=nofallback --buildtype=debugoptimized \
      -Db_ndebug=false -Db_sanitize=address,undefined -Db_lundef=false \
      -Dc_args=-fno-omit-frame-pointer -Dcpp_args=-fno-omit-frame-pointer \
      -Dglib=enabled -Dgobject=enabled -Dfreetype=enabled -Dcairo=enabled \
      -Dicu=enabled -Dgraphite2=enabled -Dpng=enabled -Dzlib=enabled \
      -Dsubset=enabled -Draster=enabled -Dvector=enabled -Dgpu=enabled \
      -Dgpu_demo=disabled -Dtests=enabled -Dutilities=enabled \
      -Ddocs=disabled -Dintrospection=disabled -Dexperimental_api=true \
    && meson compile -C build-asan -j "$(nproc)"

# Preserve test logs and binaries for the audit even if an upstream test fails.
RUN meson test -C build --print-errorlogs --num-processes 4 \
    || echo "WARNING: HarfBuzz tests reported failures; see build/meson-logs/testlog.txt"
