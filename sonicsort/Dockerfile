# syntax=docker/dockerfile:1
#
# SonicSort's own Alpine base, assembled from Alpine's package repositories by
# this fork (poslogica/docker-alpine, branch sonicsort/3.24) instead of using
# the prebuilt minirootfs tarball the official alpine image (and upstream
# docker-alpine's fetch-latest-releases.lua) ships.
#
#   docker build -f sonicsort/Dockerfile --build-arg BUILDER_BASE=sonicsort-debian:local \
#     -t sonicsort-alpine:local sonicsort
#
# How: apk.static -- Alpine's statically linked package manager, pinned by
# version and sha256 below -- installs the official image's six "world"
# packages into an empty root with --initdb. apk verifies every index and
# package signature against Alpine's public keys, vendored in keys/ and
# cross-checked against alpinelinux/aports main/alpine-keys (3.24-stable):
#   4a6a0840 9c102bcc376af1498d549b77bdbfa815ae86faa1d2d82f040e616b18ef2df2d4
#   5261cecb 12f899e55a7691225603d6fb3324940fc51cd7f133e7ead788663c2b7eecb00c
#   6165ee59 207e4696d3c05f7cb05966aee557307151f1f00217af4143c1bcaf33b8df733f
# apk only needs chroot for package scripts, which a normal docker build has,
# so -- unlike Debian's debootstrap -- no privileged pre-build step.
#
# Patching: change the package set / repositories / config here. Alpine has no
# snapshot service, so the newest packages in the branch are installed at
# build time; the weekly OS_PATCH_WEEK rebuild picks up fixes, and
# /etc/apk/sonicsort-manifest records exactly what went in.

# The builder stage only downloads and runs apk.static; SonicSort runs it on its
# own Debian (sonicsort-debian:local). Any image with curl + sha256sum works.
ARG BUILDER_BASE=debian:trixie-slim

FROM ${BUILDER_BASE} AS build

ARG ALPINE_BRANCH=v3.24
ARG ALPINE_MIRROR=https://dl-cdn.alpinelinux.org/alpine
ARG APK_TOOLS_STATIC_VERSION=3.0.8-r0
ARG APK_TOOLS_STATIC_SHA256=c8e2c88c13ba12a12269b79a3543e1190ff8c0ab0beb32b58cadfd5881c619e3
# Weekly cache-buster: SonicSort's build scripts pass the current ISO week, so
# the package install below re-runs (and picks up fixes) at least weekly.
ARG OS_PATCH_WEEK=unset

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl; \
    rm -rf /var/lib/apt/lists/*

COPY keys/ /alpine-keys/

RUN set -eux; \
    arch="$(uname -m)"; \
    curl -fsSL -o /tmp/apk-tools-static.apk \
        "${ALPINE_MIRROR}/${ALPINE_BRANCH}/main/${arch}/apk-tools-static-${APK_TOOLS_STATIC_VERSION}.apk"; \
    echo "${APK_TOOLS_STATIC_SHA256}  /tmp/apk-tools-static.apk" | sha256sum -c -; \
    tar -xzf /tmp/apk-tools-static.apk -C /tmp sbin/apk.static 2>/dev/null || [ -x /tmp/sbin/apk.static ]; \
    /tmp/sbin/apk.static --version

# Mirrors Alpine's own minirootfs build (aports scripts/genrootfs.sh, 3.24-stable):
# --no-scripts (so package install scripts don't add users such as busybox's
# klogd, whose gid 101 would collide with images that create their own, e.g.
# nginx), busybox applet symlinks created explicitly from `busybox --list-full`
# (run under chroot: it's linked against the rootfs's musl, not the builder's
# glibc; `-L` as well as `-e` because existing absolute links like
# /bin/sh -> /bin/busybox don't resolve outside the chroot), root's shadow entry locked (root:* not root:) with /etc/shadow in
# group 42 (shadow), and no apk.log.
RUN set -eux; \
    arch="$(uname -m)"; \
    /tmp/sbin/apk.static \
        --arch "$arch" \
        --keys-dir /alpine-keys \
        --repository "${ALPINE_MIRROR}/${ALPINE_BRANCH}/main" \
        --root /rootfs --initdb --no-cache --update-cache --no-scripts \
        add alpine-baselayout alpine-keys alpine-release apk-tools busybox musl-utils; \
    rm -f /rootfs/var/log/apk.log; \
    for link in $(chroot /rootfs /bin/busybox --list-full); do \
        [ -e "/rootfs/$link" ] || [ -L "/rootfs/$link" ] || ln -s /bin/busybox "/rootfs/$link"; \
    done; \
    sed -i -e 's/^root::/root:*:/' /rootfs/etc/shadow; \
    chgrp 42 /rootfs/etc/shadow; \
    printf '%s/%s/main\n%s/%s/community\n' \
        "${ALPINE_MIRROR}" "${ALPINE_BRANCH}" "${ALPINE_MIRROR}" "${ALPINE_BRANCH}" \
        > /rootfs/etc/apk/repositories; \
    /tmp/sbin/apk.static --root /rootfs info -v | sort > /rootfs/etc/apk/sonicsort-manifest; \
    rm -rf /rootfs/var/cache/apk/* /rootfs/lib/apk/db/lock /rootfs/dev/*; \
    cat /rootfs/etc/alpine-release; \
    wc -l < /rootfs/etc/apk/sonicsort-manifest

FROM scratch
COPY --from=build /rootfs/ /
CMD ["/bin/sh"]
