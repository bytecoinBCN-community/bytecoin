# Bytecoin (bcndev v3.5.1) on Ubuntu 24.04 LTS.
# Build context: this directory with ./bytecoin and ./lmdb side by side, as upstream CMake expects.
FROM ubuntu:24.04 AS build
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        build-essential cmake libboost-dev libssl-dev libudev-dev \
    && rm -rf /var/lib/apt/lists/*
# CMakeLists looks for OpenSSL in ../openssl/{include,libssl.a,libcrypto.a}; point it at the distro's static OpenSSL 3
RUN A=$(gcc -print-multiarch) && mkdir -p /src/openssl/include \
    && ln -s /usr/include/openssl /src/openssl/include/openssl \
    && ln -s /usr/include/$A/openssl/opensslconf.h /src/openssl/include/opensslconf.h \
    && ln -s /usr/lib/$A/libssl.a /src/openssl/libssl.a && ln -s /usr/lib/$A/libcrypto.a /src/openssl/libcrypto.a
COPY lmdb /src/lmdb
COPY bytecoin /src/bytecoin
ARG JOBS=2
WORKDIR /src/bytecoin/build
RUN cmake .. && make -j${JOBS} bytecoind walletd minerd tests \
    && strip ../bin/bytecoind ../bin/walletd ../bin/minerd ../bin/tests

FROM ubuntu:24.04
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        libudev1 ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && useradd --system --home-dir /var/lib/bytecoin --create-home --shell /usr/sbin/nologin bytecoin
COPY --from=build /src/bytecoin/bin/bytecoind /src/bytecoin/bin/walletd /src/bytecoin/bin/minerd /src/bytecoin/bin/tests /usr/local/bin/
USER bytecoin
VOLUME /var/lib/bytecoin
# 8080 = P2P, 8081 = node RPC, 8070 = walletd RPC
EXPOSE 8080 8081 8070
ENTRYPOINT ["bytecoind", "--data-folder=/var/lib/bytecoin"]
CMD ["--bytecoind-bind-address=0.0.0.0:8081"]
