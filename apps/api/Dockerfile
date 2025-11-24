FROM node:20-alpine3.19 AS builder
RUN apk add --no-cache g++ make py3-pip bash
ENV NX_DAEMON=false

# Disable Nx Cloud for Railway builds
ENV NX_CLOUD_ACCESS_TOKEN=""
ENV NX_CLOUD=false
ENV CI=true

WORKDIR /usr/src/app

RUN npm install -g pnpm@10.16.1 pm2 --loglevel notice

# Copy package files
COPY .npmrc .
COPY package.json .
COPY pnpm-workspace.yaml .
COPY pnpm-lock.yaml .

# Copy source directories
COPY apps/api ./apps/api
COPY libs ./libs
COPY packages ./packages
COPY enterprise ./enterprise
COPY nx.json .
COPY tsconfig.json .

# Install all dependencies
RUN pnpm install --frozen-lockfile --unsafe-perm

# Build API service
RUN NODE_ENV=production NX_DAEMON=false pnpm build:api

# Production stage
FROM node:20-alpine3.19 AS production

RUN apk add --no-cache bash
RUN npm install -g pnpm@10.16.1 pm2 --loglevel notice

USER 1000
WORKDIR /usr/src/app

# Copy built artifacts
COPY --chown=1000:1000 --from=builder /usr/src/app/apps/api/dist ./apps/api/dist
COPY --chown=1000:1000 --from=builder /usr/src/app/apps/api/package.json ./apps/api/package.json

# Copy necessary dependencies
COPY --chown=1000:1000 --from=builder /usr/src/app/node_modules ./node_modules
COPY --chown=1000:1000 --from=builder /usr/src/app/package.json ./package.json
COPY --chown=1000:1000 --from=builder /usr/src/app/pnpm-workspace.yaml ./pnpm-workspace.yaml
COPY --chown=1000:1000 --from=builder /usr/src/app/pnpm-lock.yaml ./pnpm-lock.yaml
COPY --chown=1000:1000 --from=builder /usr/src/app/packages ./packages
COPY --chown=1000:1000 --from=builder /usr/src/app/libs ./libs

ENV NEW_RELIC_NO_CONFIG_FILE=true
ENV NODE_ENV=production

WORKDIR /usr/src/app/apps/api
CMD ["pm2-runtime", "start", "dist/main.js", "-i", "max"]
