# otari: an OpenAI-compatible LLM gateway (Python/FastAPI backend, React dashboard).
# Builds the checkout at /src with dev dependencies, so the unit tests and the
# dashboard's vitest suite can run offline inside the image.
FROM node:26-slim AS node

FROM python:3.14-slim
ENV DEBIAN_FRONTEND=noninteractive \
    UV_COMPILE_BYTECODE=1 \
    COREPACK_ENABLE_DOWNLOAD_PROMPT=0
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git build-essential media-types \
    && rm -rf /var/lib/apt/lists/*
COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && npm install --global corepack@latest && corepack enable
RUN pip install --no-cache-dir uv

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Python workspace (gateway, cli/, any-search/, any-fetch/) with dev dependencies
# and the extras the published image ships with.
RUN uv sync --frozen --extra e2b --extra metrics --extra redis
ENV PATH="/src/.venv/bin:${PATH}"

# Dashboard: install from the lockfile and build the bundle the gateway serves.
RUN cd web && corepack prepare --activate && pnpm install --frozen-lockfile && pnpm run build

# Smoke the image without failing the build. Integration tests need PostgreSQL
# (Docker), so only the unit suite runs here; a few unit tests need DNS.
RUN (pytest tests/unit -q -p no:cacheprovider || echo "WARNING: unit tests reported failures")
