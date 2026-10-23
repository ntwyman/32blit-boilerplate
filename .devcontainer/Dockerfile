# 32blit development container
#
# Provides everything listed in the 32blit SDK's Linux, Emscripten and
# 32blit (device) docs:
#   - native toolchain + SDL2 for local builds
#   - gcc-arm-none-eabi for 32Blit device builds
#   - Emscripten for web builds (optional, see INSTALL_EMSDK)
#   - the 32blit SDK, checked out to /opt/32blit-sdk
#   - the 32blit tools, built from source and installed into /opt/venv

FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

ARG SDK_REPO=https://github.com/32blit/32blit-sdk
ARG SDK_REF=master
ARG TOOLS_REPO=https://github.com/32blit/32blit-tools
ARG TOOLS_REF=main
ARG INSTALL_EMSDK=true
ARG EMSDK_VERSION=latest

ENV DEBIAN_FRONTEND=noninteractive

# SDK prerequisites (see 32blit-sdk/docs/Linux.md), plus newlib for the
# device build and a debugger / ninja for VS Code
RUN apt-get update && apt-get install -y --no-install-recommends \
        git gcc g++ cmake make ninja-build gdb unzip xz-utils \
        python3 python3-pip python3-setuptools python3-venv python3-dev \
        libsdl2-dev libsdl2-image-dev libsdl2-net-dev \
        libfreetype-dev \
        gcc-arm-none-eabi libnewlib-arm-none-eabi libstdc++-arm-none-eabi-newlib \
    && rm -rf /var/lib/apt/lists/*

# Ubuntu 24.04 doesn't allow global pip installs (PEP 668), so the tools live
# in a venv that's put first on PATH for every user. The tools' setup.py
# imports pkg_resources, which setuptools 81 removed, so pin below that and
# build against the venv's setuptools rather than an isolated (latest) one
ENV VIRTUAL_ENV=/opt/venv
ENV PATH=/opt/venv/bin:$PATH
RUN python3 -m venv /opt/venv \
    && pip install --no-cache-dir --upgrade pip 'setuptools<81' wheel

# 32blit tools, built from source. bitstring 5 dropped ConstBitStream, which
# the tools still import
RUN git clone --depth 1 --branch ${TOOLS_REF} ${TOOLS_REPO} /opt/32blit-tools \
    && pip install --no-cache-dir --no-build-isolation /opt/32blit-tools/src 'bitstring<5' \
    && 32blit version

# 32blit SDK
RUN git clone --depth 1 --branch ${SDK_REF} ${SDK_REPO} /opt/32blit-sdk
ENV PATH_32BLIT_SDK=/opt/32blit-sdk

# Emscripten (see 32blit-sdk/docs/Emscripten.md)
ENV EMSDK=/opt/emsdk
RUN if [ "${INSTALL_EMSDK}" = "true" ]; then \
        git clone --depth 1 https://github.com/emscripten-core/emsdk.git ${EMSDK} \
        && cd ${EMSDK} \
        && ./emsdk install ${EMSDK_VERSION} \
        && ./emsdk activate ${EMSDK_VERSION} \
        && echo '. /opt/emsdk/emsdk_env.sh > /dev/null 2>&1' >> /etc/bash.bashrc ; \
    fi

# CMake Tools kits for cross builds; referenced from devcontainer.json
RUN printf '%s\n' \
    '[' \
    '  { "name": "32Blit", "toolchainFile": "/opt/32blit-sdk/32blit.toolchain" },' \
    '  { "name": "Emscripten", "toolchainFile": "/opt/emsdk/upstream/emscripten/cmake/Modules/Platform/Emscripten.cmake" }' \
    ']' > /opt/32blit-kits.json

# Let the default user write to the emscripten cache, the venv and the SDK
# (the 32Blit toolchain downloads its stdlibs into the SDK on first configure)
RUN chown -R vscode:vscode /opt/venv /opt/32blit-sdk \
    && if [ -d ${EMSDK} ]; then chown -R vscode:vscode ${EMSDK}; fi

# Serial access for `32blit flash` (when a device is passed through)
RUN usermod -aG dialout vscode

# Claude Code CLI (native installer, into ~/.local/bin). Its config/login goes
# in ~/.claude, which devcontainer.json mounts as a volume to survive rebuilds
ENV CLAUDE_CONFIG_DIR=/home/vscode/.claude
USER vscode
RUN mkdir -p /home/vscode/.claude \
    && curl -fsSL https://claude.ai/install.sh | bash \
    && /home/vscode/.local/bin/claude --version
ENV PATH=/home/vscode/.local/bin:$PATH
USER root
