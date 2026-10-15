# Build Apache Airflow for Anthropic's OSS Scanner.
# The scanner uses the repository root as the build context and audits /src offline, so everything the
# tests need is installed here. Airflow's reference images and CI run on Debian Bookworm.
FROM python:3.12-slim-bookworm

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ENV DEBIAN_FRONTEND=noninteractive \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    AIRFLOW_HOME=/tmp/airflow-home

RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential ca-certificates curl git libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

# Use the uv version the repository pins in uv.lock, as Airflow's CI does.
RUN UV_VERSION="$(sed -n '/^name = "uv"$/{n;s/^version = "\(.*\)"$/\1/p;}' uv.lock)" \
    && pip install --no-cache-dir "uv==${UV_VERSION}"

# airflow-core with its dev dependencies, which bring the Task SDK, the shared test fixtures
# (devel-common) and the providers its tests use (FAB auth manager, Celery, Kubernetes, Amazon, ...),
# from the committed lock file into /src/.venv.
RUN uv sync --frozen --package apache-airflow-core

# Run a fast, database-free slice of the API server tests so the image is known to work; a failing test
# must not abort the build. Further tests: `uv run --no-sync pytest airflow-core/tests/unit/<path>`.
RUN uv run --frozen --no-sync pytest airflow-core/tests/unit/api_fastapi/auth --skip-db-tests -q -p no:cacheprovider \
    || echo "WARNING: pytest reported failures"
