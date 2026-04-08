# ---- Base stage ----
FROM node:20-bookworm-slim AS base

RUN corepack enable && corepack prepare pnpm@9.15.4 --activate

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY pnpm-workspace.yaml pnpm-lock.yaml package.json tsconfig.base.json ./
COPY scripts/ scripts/
COPY packages/core/package.json packages/core/package.json
COPY packages/cli/package.json packages/cli/package.json
COPY packages/web/package.json packages/web/package.json
COPY packages/plugins/ packages/plugins/

# ---- Build stage: build everything ----
FROM base AS build

ARG NEXT_PUBLIC_TERMINAL_PORT=14800
ARG NEXT_PUBLIC_DIRECT_TERMINAL_PORT=14801
ENV NEXT_PUBLIC_TERMINAL_PORT=$NEXT_PUBLIC_TERMINAL_PORT
ENV NEXT_PUBLIC_DIRECT_TERMINAL_PORT=$NEXT_PUBLIC_DIRECT_TERMINAL_PORT

RUN pnpm install --frozen-lockfile

COPY packages/core/ packages/core/
COPY packages/cli/ packages/cli/
COPY packages/web/ packages/web/

# Build core + plugins + cli + web
RUN pnpm --filter @composio/ao-cli... build
RUN pnpm --filter @composio/ao-web... build

# Deploy CLI to isolated directory with prod deps only
RUN pnpm deploy --filter=@composio/ao-cli --prod /app/cli-prod

# ---- Production stage ----
FROM node:20-bookworm-slim AS prod

RUN corepack enable && corepack prepare pnpm@9.15.4 --activate

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-pip git gh tmux tini curl jq bubblewrap \
    openssh-client ca-certificates \
    && update-ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy web (needs full packages + node_modules for pnpm workspace)
COPY --from=build --chown=node:node /app/packages packages
COPY --from=build --chown=node:node /app/node_modules node_modules
COPY --from=build --chown=node:node /app/package.json package.json

# Copy CLI standalone build
COPY --from=build --chown=node:node /app/cli-prod /app/cli
COPY --chown=node:node scripts/ scripts/

# ao CLI symlink
RUN ln -sf /app/cli/dist/index.js /usr/local/bin/ao && chmod +x /usr/local/bin/ao

# Install codex globally
RUN npm install -g @openai/codex

USER node

EXPOSE 3000 14800 14801

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/ || exit 1

ENTRYPOINT ["tini", "--"]

CMD ["sh", "-c", "ao & cd /app/packages/web && pnpm start:all"]