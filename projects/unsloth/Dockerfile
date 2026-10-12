# Unsloth: Python library (unsloth/, unsloth_cli/) and Unsloth Studio (studio/backend FastAPI, studio/frontend React).
# CPU-only image mirroring the "Repo tests (CPU)" CI job; the test suite spoofs CUDA where the import chain needs it.
FROM python:3.12-bookworm
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_NO_CACHE_DIR=1 \
    UNSLOTH_COMPILE_DISABLE=1 \
    PYTHONPATH=/src/studio
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential curl \
    && rm -rf /var/lib/apt/lists/*

# Node 22 for the frontend tests (node --experimental-strip-types); Debian's nodejs is too old.
COPY --from=node:22-bookworm /usr/local/bin/node /usr/local/bin/node
COPY --from=node:22-bookworm /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
    && node --version && npm --version

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN python -m pip install --upgrade pip uv \
    && pip install -r studio/backend/requirements/studio.txt \
    && pip install python-multipart aiofiles sqlalchemy cryptography psutil pyyaml jinja2 mammoth unpdf requests typer \
         'numpy<3' pytest pytest-asyncio pytest-xdist pytest-timeout httpx \
    && pip install --index-url https://download.pytorch.org/whl/cpu --extra-index-url https://pypi.org/simple \
         'torch>=2.4,<2.11' 'torchvision<0.26' 'torchaudio<2.11' \
    && pip install 'transformers>=4.51,<=5.17.0' sentence-transformers 'bitsandbytes>=0.45' \
    && pip install "unsloth_zoo @ git+https://github.com/unslothai/unsloth-zoo" \
    && pip install -e . --no-deps

# Unsloth Studio installed the supported way from this checkout (the Studio venv and the built web UI), so a real
# server can be started offline. No torch in the Studio venv: chat, auth, uploads and the API all run without it.
# The password is a throwaway for this offline sandbox only.
RUN UNSLOTH_NO_TORCH=1 \
    UNSLOTH_SKIP_AUTOSTART=1 \
    UNSLOTH_STUDIO_HOME=/workspace/unsloth_studio \
    UNSLOTH_STUDIO_PASSWORD=scanner-sandbox-only \
    ./install.sh --local
ENV UNSLOTH_STUDIO_HOME=/workspace/unsloth_studio \
    UNSLOTH_STUDIO_PASSWORD=scanner-sandbox-only

# Frontend dependencies, so its node tests run offline.
RUN cd studio/frontend && (npm ci --no-audit --no-fund || echo "WARNING: npm ci failed; frontend tests unavailable")

# Cheap smoke run so the image is known to work; failures do not abort the build.
RUN (python -m pytest studio/backend/tests/test_middleware.py studio/backend/tests/test_rag_native_drop_upload.py -q --tb=short -p no:cacheprovider \
      || echo "WARNING: smoke tests reported failures")
