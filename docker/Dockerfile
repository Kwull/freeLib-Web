# syntax=docker/dockerfile:1.7
# The web app is platform-independent: build it once on the build machine.
FROM --platform=$BUILDPLATFORM node:22-trixie-slim AS web
RUN npm install -g pnpm@10
WORKDIR /build
COPY web/package.json web/pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY web/ ./
RUN pnpm build

# The runtime is Debian trixie: build against the same glibc. Rust is
# cross-compiled on the build machine instead of running the compiler under
# QEMU emulation, which makes the arm64 image build take minutes, not hours.
FROM --platform=$BUILDPLATFORM rust:1-trixie AS build
ARG TARGETARCH
RUN case "$TARGETARCH" in \
      amd64) echo x86_64-unknown-linux-gnu > /rust-target; pkgs="gcc-x86-64-linux-gnu libc6-dev-amd64-cross" ;; \
      arm64) echo aarch64-unknown-linux-gnu > /rust-target; pkgs="gcc-aarch64-linux-gnu libc6-dev-arm64-cross" ;; \
      *) echo "unsupported TARGETARCH=$TARGETARCH" >&2; exit 1 ;; \
    esac \
    && if [ "$(dpkg --print-architecture)" != "$TARGETARCH" ]; then \
         apt-get update && apt-get install -y --no-install-recommends $pkgs && rm -rf /var/lib/apt/lists/*; \
       fi \
    && rustup target add "$(cat /rust-target)"
ENV CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc \
    CC_aarch64_unknown_linux_gnu=aarch64-linux-gnu-gcc \
    AR_aarch64_unknown_linux_gnu=aarch64-linux-gnu-ar \
    CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=x86_64-linux-gnu-gcc \
    CC_x86_64_unknown_linux_gnu=x86_64-linux-gnu-gcc \
    AR_x86_64_unknown_linux_gnu=x86_64-linux-gnu-ar
COPY server/ /src/server/
COPY --from=web /build/dist /src/web/dist
WORKDIR /src/server
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/src/server/target \
    target="$(cat /rust-target)" \
    && cargo build --release --locked -p freelib-server --target "$target" \
    && cp "/src/server/target/$target/release/freelib-server" /tmp/freelib-server

# CVE-2026-111497: an out-of-bounds write in Brotli's C decoder (1.0.3 through
# 1.2.0), exploited in the wild. Calibre reaches it through freetype (WOFF2 fonts
# embedded in books). Upstream fixed it in 1.2.1; trixie still ships libbrotli1
# 1.1.0, so build 1.2.1 here as a drop-in libbrotli1 package (same .so.1 soname).
# Its version sorts above any 1.1.0-*+deb13uN, so apt keeps it. Drop this stage
# once trixie's libbrotli1 carries the fix.
FROM debian:trixie-slim AS brotli
ARG BROTLI_VERSION=1.2.1
# The commit tagged v1.2.1 in github.com/google/brotli
ARG BROTLI_COMMIT=db24a98aed38a16de78cc05ca15cdd8dc2a0d92d
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates cmake gcc git libc6-dev make \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /src
RUN git init -q . \
    && git fetch -q --depth 1 https://github.com/google/brotli "${BROTLI_COMMIT}" \
    && git checkout -q FETCH_HEAD \
    && test "$(git rev-parse HEAD)" = "${BROTLI_COMMIT}" \
    && multiarch="$(gcc -print-multiarch)" \
    && cmake -S . -B out -DCMAKE_BUILD_TYPE=Release -DBROTLI_DISABLE_TESTS=ON \
       -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR="lib/$multiarch" \
    && cmake --build out -j"$(nproc)" \
    && DESTDIR=/stage cmake --install out \
    && mkdir -p /pkg/DEBIAN "/pkg/usr/lib/$multiarch" /pkg/usr/share/doc/libbrotli1 /out \
    && cp -a /stage/usr/lib/"$multiarch"/libbrotli*.so.* "/pkg/usr/lib/$multiarch/" \
    && cp LICENSE /pkg/usr/share/doc/libbrotli1/copyright \
    && printf '%s\n' \
       'Package: libbrotli1' \
       'Source: brotli' \
       "Version: ${BROTLI_VERSION}-0freelib1" \
       "Architecture: $(dpkg --print-architecture)" \
       'Multi-Arch: same' \
       'Maintainer: freeLib <noreply@localhost>' \
       'Depends: libc6' \
       'Section: libs' \
       'Priority: optional' \
       "Description: Brotli library ${BROTLI_VERSION} (upstream build, fixes CVE-2026-111497)" \
       > /pkg/DEBIAN/control \
    && dpkg-deb --root-owner-group --build /pkg /out/libbrotli1.deb

FROM debian:trixie-slim AS slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    tini \
    && rm -rf /var/lib/apt/lists/*
# Upgrade libbrotli1 to the fixed build wherever a package pulled it in.
RUN --mount=type=bind,from=brotli,source=/out,target=/mnt/brotli \
    if dpkg-query -W libbrotli1 >/dev/null 2>&1; then dpkg -i /mnt/brotli/libbrotli1.deb; fi
RUN useradd -m -u 1000 freelib \
    && mkdir -p /data /cache /export /books \
    && chown freelib:freelib /data /cache /export
COPY --from=build /tmp/freelib-server /usr/local/bin/freelib-server
COPY docker/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/freelib-server /usr/local/bin/entrypoint.sh
# The container starts as root only long enough for entrypoint.sh to give the
# volumes to PUID:PGID; the server itself runs as that unprivileged user.
ENV PUID=1000 \
    PGID=1000 \
    FREELIB_PORT=8080 \
    FREELIB_DATA_DIR=/data \
    FREELIB_BOOKS_DIR=/books \
    FREELIB_CACHE_DIR=/cache \
    FREELIB_EXPORT_DIR=/export
VOLUME /data /cache /export
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD freelib-server healthcheck || exit 1
ENTRYPOINT ["tini", "--", "/usr/local/bin/entrypoint.sh"]
CMD ["freelib-server"]

# trixie ships Calibre 8.x; the server refuses Calibre < 6.19 (CVE-2023-46303, bookworm has 6.13)
FROM slim AS full
RUN apt-get update && apt-get install -y --no-install-recommends \
    calibre \
    fonts-dejavu-core \
    fonts-liberation \
    && rm -rf /var/lib/apt/lists/*
# Calibre (via freetype) pulls in trixie's vulnerable libbrotli1; replace it.
RUN --mount=type=bind,from=brotli,source=/out,target=/mnt/brotli \
    dpkg -i /mnt/brotli/libbrotli1.deb \
    && dpkg --compare-versions "$(dpkg-query -W -f='${Version}' libbrotli1)" ge 1.2.1
ENV QT_QPA_PLATFORM=offscreen \
    FREELIB_CALIBRE=/usr/bin/ebook-convert
