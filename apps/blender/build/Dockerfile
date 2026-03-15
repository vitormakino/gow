# syntax=docker/dockerfile:1.4
ARG BASE_APP_IMAGE

# hadolint ignore=DL3006
FROM ${BASE_APP_IMAGE}

ARG DEBIAN_FRONTEND=noninteractive

ARG REQUIRED_PACKAGES=" \
    blender \
    fonts-dejavu-core \
    libxkbcommon0 \
    libdbus-1-3 \
"

RUN <<_INSTALL_PACKAGES
apt-get update -y
apt-get install -y --no-install-recommends $REQUIRED_PACKAGES
apt-get autoremove -y
rm -rf /var/lib/apt/lists/*
_INSTALL_PACKAGES

COPY --chmod=777 scripts/startup.sh /opt/gow/startup-app.sh

ENV XDG_RUNTIME_DIR=/tmp/.X11-unix

ARG IMAGE_SOURCE
LABEL org.opencontainers.image.source=$IMAGE_SOURCE