ARG BUILD_FROM
FROM $BUILD_FROM AS builder-base

# hadolint ignore=DL3009
RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       git \
       patch \
       gcc g++ \
       make \
       pkg-config

FROM builder-base AS builder-ttyd

ARG LIBWEBSOCKETS_VERSION
ARG TTYD_VERSION

COPY cmake-remove-duplicate-LwsCheckRequirements.patch /usr/src

WORKDIR /usr/src

RUN \
    set -x \
    && apt-get install -y --no-install-recommends \
       cmake \
       libuv1-dev \
       libc-dev \
       libssl-dev \
       zlib1g-dev \
       libjson-c-dev \
    && git clone --branch "v${LIBWEBSOCKETS_VERSION}" --depth=1 \
        https://github.com/warmcat/libwebsockets \
    && cd libwebsockets \
    && patch -p 1 < /usr/src/cmake-remove-duplicate-LwsCheckRequirements.patch \
    && mkdir -p build \
    && cd build \
    && cmake .. \
        -DCMAKE_BUILD_TYPE=MinSizeRel \
        -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DCMAKE_VERBOSE_MAKEFILE=TRUE \
        -DLWS_IPV6=ON \
        -DLWS_STATIC_PIC=ON \
        -DLWS_UNIX_SOCK=OFF \
        -DLWS_WITH_LIBUV=ON \
        -DLWS_WITH_SHARED=ON \
        -DLWS_WITHOUT_TESTAPPS=ON \
    && make \
    && make install \
    && make DESTDIR=./install install

RUN \
    cd /usr/src \
    && git clone --branch main --single-branch \
        https://github.com/tsl0922/ttyd \
    && cd ttyd \
    && git checkout "${TTYD_VERSION}" \
    && mkdir -p build \
    && cd build \
    && cmake .. \
        -DCMAKE_BUILD_TYPE=MinSizeRel \
        -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DCMAKE_VERBOSE_MAKEFILE=TRUE \
    && make \
    && make DESTDIR=./install install

FROM builder-base AS builder-chip

ARG BUILD_ARCH=amd64
ARG CHIP_VERSION=a17c005209fbde737edb97b0f51ea63472c166aa

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

COPY 0001-Use-data-as-platform-storage-location.patch /usr/src
COPY 0002-Use-data-as-storage-location-for-chip-tool.patch /usr/src
COPY 0001-Avoid-python_runner.py-deadlock.patch /usr/src
COPY 0001-Avoid-polluting-Pigweed-environment-namespace.patch /usr/src

WORKDIR /usr/src

# Install and build CHIP Python Controller
# hadolint ignore=SC1091
RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       protobuf-compiler \
       libprotobuf-dev \
       gn \
       libssl-dev \
       libdbus-1-dev \
       libglib2.0-dev \
       libreadline-dev \
       ninja-build \
       python3-venv \
       python3-dev \
       python3-pip \
       python3-dbus \
       python3-click \
       python3-coloredlogs \
       python3-cryptography \
       unzip \
       libgirepository1.0-dev \
       libcairo2-dev \
    && git clone --depth 1 -b master \
       https://github.com/project-chip/connectedhomeip \
    && cd connectedhomeip \
    && git fetch origin ${CHIP_VERSION} \
    && git checkout ${CHIP_VERSION} \
    && git submodule update --init \
    && patch -p1 < /usr/src/0001-Use-data-as-platform-storage-location.patch \
    && patch -p1 < /usr/src/0002-Use-data-as-storage-location-for-chip-tool.patch \
    && patch -p1 < /usr/src/0001-Avoid-polluting-Pigweed-environment-namespace.patch \
    && ( \
           cd third_party/pigweed/repo/ \
           && patch -p1 < /usr/src/0001-Avoid-python_runner.py-deadlock.patch \
       ) \
    && export PW_CONFIG_FILE=scripts/environment_no_cipd.json \
    && source scripts/bootstrap.sh \
    && if [ "${BUILD_ARCH}" = "amd64" ]; \
       then \
           EXAMPLE_PREFIX="linux-x64-"; \
       elif [ "${BUILD_ARCH}" = "aarch64" ]; \
       then \
           EXAMPLE_PREFIX="linux-arm64-"; \
       else \
           exit 1; \
       fi \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}chip-tool build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}chip-cert build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}all-clusters-no-ble-no-wifi build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}bridge build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}light build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}lock build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}thermostat build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}tv-app build \
    && scripts/build/build_examples.py --target ${EXAMPLE_PREFIX}tv-casting-app build

FROM $BUILD_FROM

RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       libuv1 \
       openssl \
       zlib1g \
       libjson-c5 \
       libdbus-1-3 \
       libglib2.0 \
       tmux \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/src/*

COPY --from=builder-ttyd /usr/src/libwebsockets/build/install /
COPY --from=builder-ttyd /usr/src/ttyd/build/install /

RUN ldconfig

WORKDIR /root

COPY --from=builder-chip /usr/src/connectedhomeip/credentials /root/credentials
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-chip-tool/chip-tool /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-chip-cert/chip-cert /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-all-clusters-no-ble-no-wifi/address-resolve-tool /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-all-clusters-no-ble-no-wifi/chip-all-clusters-app /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-bridge/chip-bridge-app /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-light/chip-lighting-app /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-lock/chip-lock-app /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-thermostat/thermostat-app /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-tv-app/chip-tv-app /usr/local/bin
COPY --from=builder-chip /usr/src/connectedhomeip/out/linux-*-tv-casting-app/chip-tv-casting-app /usr/local/bin

COPY rootfs /
