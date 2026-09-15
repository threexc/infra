FROM alpine:latest
ARG VERSION
# One of x64 or arm64 or ppc64le or riscv64
ARG ARCH
# The sha256sum for the node source tarball
ARG SHA256SUM

RUN mkdir -p /build
WORKDIR /build

# binutils-gold has no riscv64 support in Alpine; falls back to ld.bfd there
RUN apk add --no-cache curl g++ gcc gnupg libgcc linux-headers make ninja python3 libstdc++ patch && \
    if [ "$ARCH" != "riscv64" ]; then apk add --no-cache binutils-gold; fi
RUN mkdir -p /out/bin

RUN curl -O https://nodejs.org/dist/$VERSION/node-$VERSION.tar.gz
RUN echo "$SHA256SUM  node-$VERSION.tar.gz" | sha256sum -c -
RUN tar -zxvf node-$VERSION.tar.gz

# Apply https://github.com/abseil/abseil-cpp/pull/1918
# Needed to build on ppc64le
COPY frequency-musl-ppc64le.patch /tmp/
RUN if [ "$VERSION" = "v24.14.1" ]; then \
    cd node-$VERSION && \
    patch -p1 < /tmp/frequency-musl-ppc64le.patch; \
fi

RUN cd node-$VERSION && \
    case "$ARCH" in \
      ppc64le) CPU=ppc64 ;; \
      *) CPU="$ARCH" ;; \
    esac && \
    OPENSSL_ASM_FLAG="" && \
    # '-m64' doesn't work for riscv64, so drop asm
    if [ "$ARCH" = "riscv64" ]; then OPENSSL_ASM_FLAG="--openssl-no-asm"; fi && \
    ./configure --dest-cpu=$CPU --fully-static --ninja --without-inspector --with-intl=none --without-npm --without-corepack $OPENSSL_ASM_FLAG && \
    ninja -j$(nproc) -C out/Release node

RUN cp /build/node-$VERSION/LICENSE /out/LICENSE && cp /build/node-$VERSION/out/Release/node /out/bin/node

FROM scratch
COPY --from=0 /out/ /out
