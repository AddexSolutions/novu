FROM node:20-alpine3.19 AS dev_base
RUN apk --update --no-cache add curl g++ make py3-pip
ENV NX_DAEMON=false


RUN npm i pm2 -g
RUN npm --no-update-notifier --no-fund --global install pnpm@10.16.1
RUN pnpm --version


USER 1000
WORKDIR /usr/src/app

# ------- DEV BUILD ----------
FROM dev_base AS dev
ARG PACKAGE_PATH

# Disable Nx Cloud for Railway builds
ENV NX_CLOUD_ACCESS_TOKEN=""
ENV NX_DAEMON=false
ENV NX_CLOUD=false
ENV CI=true
ENV NX_SKIP_NX_CACHE=true
ENV NX_CLOUD_DISABLE=true

# These directories may not exist in build context - skip copying

RUN if [ -n "${BULL_MQ_PRO_NPM_TOKEN}" ] ; then echo 'Building with Enterprise Edition of Novu'; rm -f .npmrc ; cp .npmrc-cloud .npmrc ; fi

RUN pnpm install --filter "novuhq" --filter "{${PACKAGE_PATH}}..."\
 --frozen-lockfile\
 --unsafe-perm\
 --reporter=silent

RUN NODE_ENV=production NX_DAEMON=false pnpm run build:worker

WORKDIR /usr/src/app/apps/worker

RUN cp src/dotenvcreate.mjs dist/dotenvcreate.mjs
RUN cp src/.example.env dist/.env
RUN cp src/.env.test dist/.env.test
RUN cp src/.env.development dist/.env.development
RUN cp src/.env.production dist/.env.production

WORKDIR /usr/src/app

# ------- ASSETS BUILD ----------
FROM dev AS assets

WORKDIR /usr/src/app

# Remove all dependencies so later we can only install prod dependencies without devDependencies
RUN rm -rf node_modules && pnpm recursive exec -- rm -rf ./src ./node_modules

# ------- PRODUCTION BUILD ----------
FROM dev_base AS prod
ARG PACKAGE_PATH

ENV CI=true

WORKDIR /usr/src/app

# Removed: COPY ./meta . (directory doesn't exist in Railway build context)

# Get the build artifacts that only include dist folders
COPY --chown=1000:1000 --from=assets /usr/src/app .

RUN pnpm install --filter "{${PACKAGE_PATH}}..." \
 --frozen-lockfile \
 --unsafe-perm \
 --reporter=silent

ENV NEW_RELIC_NO_CONFIG_FILE=true

WORKDIR /usr/src/app/apps/worker
ENTRYPOINT [ "sh", "-c", "node dist/dotenvcreate.mjs -s=$SECRET_NAME -r=$NOVU_REGION -e=$NOVU_ENTERPRISE -v=$NODE_ENV -h=$IS_SELF_HOSTED && pm2-runtime start dist/main.js -i max" ]
