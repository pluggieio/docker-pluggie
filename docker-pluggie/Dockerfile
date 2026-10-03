# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION
ARG BUILD_FROM
ARG S6_OVERLAY_VERSION
ARG JEMALLOC_VERSION
ARG BASHIO_VERSION
ARG TEMPIO_VERSION

FROM ${BUILD_FROM:-x} AS builder

# Re-set build arguments after FROM
ARG BUILD_ARCH
ARG S6_OVERLAY_VERSION
ARG JEMALLOC_VERSION
ARG BASHIO_VERSION
ARG TEMPIO_VERSION

# LetsEncrypt
ARG CRYPTOGRAPHY_VERSION
ARG CERTBOT_VERSION
ARG ACME_VERSION

# WireGuard
ARG WIREGUARDTOOLS_VERSION

# Set initial S6 verbosity
ENV S6_VERBOSITY=0 \
    S6_LOGGING_SCRIPT=0 \
    S6_RC_VERBOSITY=0 \
    S6_CMD_RECEIVE_LOGGING=0 \
    LANG="C.UTF-8" \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=2 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_CMD_WAIT_FOR_SERVICES=1 \
    S6_SERVICES_READYTIME=50

# Set shell
SHELL ["/bin/ash", "-o", "pipefail", "-c"]

# Install build dependencies
RUN set -x \
    && apk add --no-cache \
        python3 \
        py3-pip \
        build-base \
        python3-dev \
        musl-dev \
        linux-headers \
        gcc \
        curl \
        libffi-dev \
        openssl-dev \
        pkgconfig \
        cargo \
        rust \
    # Install Python packages to a temporary directory
    && mkdir -p /wheels \
    && pip3 wheel --wheel-dir=/wheels --no-cache-dir \
        cryptography==${CRYPTOGRAPHY_VERSION} \
        certbot==${CERTBOT_VERSION} \
        acme==${ACME_VERSION} \
        wireguard-tools==${WIREGUARDTOOLS_VERSION}

# Determine S6 architecture
RUN if [ "$BUILD_ARCH" = "armv7" ]; then \
        S6_ARCH="arm"; \
    elif [ "$BUILD_ARCH" = "i386" ]; then \
        S6_ARCH="i686"; \
    elif [ "$BUILD_ARCH" = "amd64" ]; then \
        S6_ARCH="x86_64"; \
    else \
    # aarch64, armhf
        S6_ARCH="$BUILD_ARCH"; \
    fi \
    && echo $S6_ARCH > /s6_arch \
    # Download S6 overlay files
    && mkdir -p /s6-overlay \
    && curl -L -f -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz" -o /s6-overlay/s6-overlay-arch.tar.xz \
    && curl -L -f -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz" -o /s6-overlay/s6-overlay-noarch.tar.xz \
    && curl -L -f -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-symlinks-arch.tar.xz" -o /s6-overlay/s6-overlay-symlinks-arch.tar.xz \
    && curl -L -f -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-symlinks-noarch.tar.xz" -o /s6-overlay/s6-overlay-symlinks-noarch.tar.xz \
    # Download tempio
    && mkdir -p /tempio \
    && curl -L -f -s -o /tempio/tempio \
        "https://github.com/home-assistant/tempio/releases/download/${TEMPIO_VERSION}/tempio_${BUILD_ARCH}"


FROM ${BUILD_FROM:-x} AS runtime

# Inherit build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION
ARG S6_OVERLAY_VERSION
ARG BASHIO_VERSION

# Container version
ENV PLUGGIE_VERSION=${BUILD_VERSION}

# Set environment variables
ENV S6_VERBOSITY=0 \
    S6_LOGGING_SCRIPT=0 \
    S6_RC_VERBOSITY=0 \
    S6_CMD_RECEIVE_LOGGING=0 \
    LANG="C.UTF-8" \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=2 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_CMD_WAIT_FOR_SERVICES=1 \
    S6_SERVICES_READYTIME=50

# Set shell
SHELL ["/bin/ash", "-o", "pipefail", "-c"]

# Install runtime dependencies
RUN set -x \
    && apk add --no-cache \
        nginx \
        nginx-mod-http-brotli \
        wireguard-tools \
        openresolv \
        openssl \
        bash \
        bind-tools \
        ca-certificates \
        curl \
        jq \
        libstdc++ \
        tzdata \
        xz \
        python3 \
        py3-pip \
    # Create required directories
    && mkdir -p /etc/fix-attrs.d \
    && mkdir -p /etc/services.d \
    # Create symlink for different python binary location per platform (docker/ha)
    && if [ -e /usr/bin/python ] && [ ! -e /usr/local/bin/python ]; then \
            ln -s /usr/bin/python /usr/local/bin/python; \
        elif [ -e /usr/local/bin/python ] && [ ! -e /usr/bin/python ]; then \
            ln -s /usr/local/bin/python /usr/bin/python; \
        fi

# Copy S6 overlay from builder stage
COPY --from=builder /s6_arch /s6_arch
COPY --from=builder /s6-overlay /s6-overlay
RUN S6_ARCH=$(cat /s6_arch) \
    && tar Jxvf /s6-overlay/s6-overlay-arch.tar.xz -C / \
    && tar Jxvf /s6-overlay/s6-overlay-noarch.tar.xz -C / \
    && tar Jxvf /s6-overlay/s6-overlay-symlinks-arch.tar.xz -C / \
    && tar Jxvf /s6-overlay/s6-overlay-symlinks-noarch.tar.xz -C / \
    && rm -rf /s6-overlay /s6_arch

# Copy tempio from builder stage
COPY --from=builder /tempio/tempio /usr/bin/tempio
RUN chmod a+x /usr/bin/tempio

# Install Python packages from wheels
COPY --from=builder /wheels /wheels
RUN pip3 install --no-cache-dir --break-system-packages --root-user-action=ignore /wheels/*.whl \
    && rm -rf /wheels \
    # Bashio installation
    && ln -s /usr/lib/bashio/bashio /usr/bin/bashio \
    # Remove redundant python files and internal python cache
    && rm -rf /usr/lib/python3*/site-packages/pip* \
    && find /usr/lib/python3*/site-packages -name "tests" -type d -exec rm -rf {} \; 2>/dev/null || true \
    && find /usr/lib/python3*/site-packages -name "*.py" | while read f; do if [ -f "${f}c" ]; then rm "$f"; fi; done

# Copy root filesystem
COPY rootfs /

# Change permissions
RUN chmod +x /usr/local/bin/* /etc/cont-init.d/*.sh /etc/services.d/*/run

# S6-Overlay
WORKDIR /
ENTRYPOINT ["/init"]

# Docker labels
LABEL \
    maintainer="Pluggie <support@pluggie.net>" \
    org.opencontainers.image.title="Pluggie" \
    org.opencontainers.image.description="Make your device visible! (Docker Edition)" \
    org.opencontainers.image.vendor="Pluggie" \
    org.opencontainers.image.authors="Pluggie <support@pluggie.net>" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://addons.community" \
    org.opencontainers.image.source="https://github.com/pluggieio/docker-pluggie" \
    org.opencontainers.image.documentation="https://github.com/pluggieio/docker-pluggie/blob/master/README.md"
