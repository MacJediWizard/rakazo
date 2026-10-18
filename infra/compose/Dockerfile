FROM node:22-bookworm@sha256:0557ac14e0d45d02ed563067b82856ca5e7aa3437fa28d98d4350ea9c3d9494a
RUN corepack enable
WORKDIR /app
# Manifests land before the source tree so editing a source file does not re-run the
# install. pnpm gets a partial workspace: every package listed in pnpm-workspace.yaml
# needs its manifest below, or --frozen-lockfile fails. The store is a cache mount, so
# a lockfile change reuses whatever was already downloaded.
COPY --chmod=755 package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY --chmod=755 patches patches
COPY --chmod=755 apps/api/package.json apps/api/
COPY --chmod=755 apps/desktop/package.json apps/desktop/
COPY --chmod=755 apps/mobile/package.json apps/mobile/
COPY --chmod=755 apps/web/package.json apps/web/
COPY --chmod=755 apps/worker/package.json apps/worker/
COPY --chmod=755 apps/www/package.json apps/www/
COPY --chmod=755 packages/adapter-kit/package.json packages/adapter-kit/
COPY --chmod=755 packages/adapters/package.json packages/adapters/
COPY --chmod=755 packages/auth/package.json packages/auth/
COPY --chmod=755 packages/chat-ui/package.json packages/chat-ui/
COPY --chmod=755 packages/contracts/package.json packages/contracts/
COPY --chmod=755 packages/core/package.json packages/core/
COPY --chmod=755 packages/db/package.json packages/db/
COPY --chmod=755 packages/logging/package.json packages/logging/
COPY --chmod=755 packages/memory/package.json packages/memory/
COPY --chmod=755 packages/testkit/package.json packages/testkit/
COPY --chmod=755 packages/ui-tokens/package.json packages/ui-tokens/
COPY --chmod=755 packages/ui-web/package.json packages/ui-web/
COPY --chmod=755 infra/sandboxes/supervisor/package.json infra/sandboxes/supervisor/
COPY --chmod=755 infra/updater/package.json infra/updater/
RUN --mount=type=cache,id=pnpm-store,target=/pnpm/store pnpm install --frozen-lockfile --store-dir=/pnpm/store
# Normalize modes: a restrictive host umask (e.g. 077 on hardened hosts) would otherwise
# leave the copied tree unreadable to USER node at runtime.
COPY --chmod=755 . .
RUN pnpm --filter @rakazo/db generate
# Browser error reports are symbolicated by debug ID: hidden maps are uploaded when the build has
# collector credentials, then removed so the image never serves them. Credentials and the collector
# address arrive as build secrets, so neither lands in image history. BuildKit leaves secrets out of
# the cache key, so a build that should upload passes a fresh SOURCEMAP_UPLOAD_ID to rerun this step.
ARG SOURCEMAP_UPLOAD_ID
RUN --mount=type=secret,id=sentry_auth_token,required=false \
    --mount=type=secret,id=sentry_url,required=false \
    --mount=type=secret,id=sentry_org,required=false \
    --mount=type=secret,id=sentry_project,required=false \
    set -e; \
    RAKAZO_ALLOW_DEV_SECRETS=1 WEB_SOURCEMAPS=hidden pnpm --filter @rakazo/web build; \
    cd apps/web; \
    node_modules/.bin/sentry-cli sourcemaps inject dist >/dev/null; \
    if [ -s /run/secrets/sentry_auth_token ]; then \
      SENTRY_AUTH_TOKEN="$(cat /run/secrets/sentry_auth_token)" \
      SENTRY_URL="$(cat /run/secrets/sentry_url)" \
      SENTRY_ORG="$(cat /run/secrets/sentry_org)" \
      SENTRY_PROJECT="$(cat /run/secrets/sentry_project)" \
      node_modules/.bin/sentry-cli sourcemaps upload dist \
        || echo "warning: source map upload failed; browser stack traces stay minified" >&2; \
    fi; \
    find dist -name '*.map' -delete
RUN install -d -m 750 -o node -g node /data /app/apps/web/node_modules/.vite-temp
ARG GIT_SHA
ENV NODE_ENV=production
ENV GIT_SHA=$GIT_SHA
EXPOSE 3100 5173 7091
USER node
CMD ["pnpm", "--filter", "@rakazo/api", "start"]
