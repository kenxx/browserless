ARG VERSION=latest

# ============================================================
# Builder stage: compile TypeScript, generate static + extension
# assets, install the debugger. Inherits the full toolchain
# (devDependencies + optionalDependencies) from the base image.
# ============================================================
FROM ghcr.io/browserless/base:$VERSION AS builder
LABEL org.opencontainers.image.source=https://github.com/browserless/browserless

COPY src src/

RUN npm run build && \
    npm run build:function && \
    npm run install:debugger

# ============================================================
# Runtime stage: production-only deps + Chromium + built artifacts.
# `npm prune --omit=dev` strips ~100 MB of build/lint/test tooling.
# ============================================================
FROM ghcr.io/browserless/base:$VERSION
LABEL org.opencontainers.image.source=https://github.com/browserless/browserless

# Don't let the base image's `npm install` trigger an auto-update of
ENV CLOAKBROWSER_AUTO_UPDATE=false
ENV CLOAKBROWSER_CACHE_DIR=/opt/cloakbrowser

# Drop devDependencies — the base image installs the full tree so the
# builder stage can compile, but the runtime image only needs runtime deps.
RUN npm prune --omit=dev

# NOTE it's important to not use npx playwright-core here since it'll likely install
# a more recent version than we potentially have in our own package.json
RUN ./node_modules/playwright-core/cli.js install-deps chromium && \
    ./node_modules/playwright-core/cli.js install --with-deps ffmpeg && \
    mkdir -p "$CLOAKBROWSER_CACHE_DIR" && \
    npm install cloakbrowser && \
    npx cloakbrowser install && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Pull built artifacts from builder. `static/` and `extensions/` are
# enriched by `npm run build` (adblock filters, devtools, function
# bundle, debugger payload).
COPY --from=builder $APP_DIR/build ./build
COPY --from=builder $APP_DIR/static ./static
COPY --from=builder $APP_DIR/extensions ./extensions

RUN chown -R blessuser:blessuser $APP_DIR

RUN os="$(uname -s | tr '[:upper:]' '[:lower:]')" && \
    version="$(cat "$CLOAKBROWSER_CACHE_DIR/latest_version_linux-arm64" 2>/dev/null || ls -1d "$CLOAKBROWSER_CACHE_DIR"/chromium-* | sed 's|.*/chromium-||' | sort -V | tail -n1)" && \
    revision="$(node -p 'require("/usr/src/app/node_modules/playwright-core/browsers.json").browsers.find(b=>b.name==="chromium").revision')" && \
    # to fake headless shell
    ln -sfn chrome "$CLOAKBROWSER_CACHE_DIR/chromium-$version/headless_shell" && \
    mkdir -p "$PLAYWRIGHT_BROWSERS_PATH/chromium-$revision" "$PLAYWRIGHT_BROWSERS_PATH/chromium_headless_shell-$revision" && \
    # link to cloakbrowser's chromium so that playwright can find it. 
    # This is needed since cloakbrowser's chromium is patched and won't be compatible with the revision-based path that playwright expects
    ln -sfn "$CLOAKBROWSER_CACHE_DIR/chromium-$version" "$PLAYWRIGHT_BROWSERS_PATH/chromium-$revision/chrome-$os" && \
    ln -sfn "$CLOAKBROWSER_CACHE_DIR/chromium-$version" "$PLAYWRIGHT_BROWSERS_PATH/chromium_headless_shell-$revision/chrome-$os"

USER blessuser

CMD ["./scripts/start.sh"]
