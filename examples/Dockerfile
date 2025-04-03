#
# Dockerfile for Devcoind
#
# version: 0.3.0
#

FROM debian:bookworm-slim AS builder
LABEL org.opencontainers.image.authors="Fernando Paredes Garcia <fernando@develcuy.com>"

# Update packages and install build dependencies
RUN apt-get update && apt-get install --no-install-recommends -y git wget ca-certificates \
                           build-essential libtool autotools-dev automake pkg-config bsdmainutils python3 \
                           libevent-dev libboost-dev libboost-system-dev libboost-filesystem-dev libboost-test-dev libcurl4-openssl-dev \
    && rm -rf /var/lib/apt/lists/*;
ENV SRC=/usr/local/src
ENV BDBVER=db-4.8.30.NC
WORKDIR $SRC
# Download and Compile Berkeley DB 4.8
RUN set -ex; \
        mkdir -p $SRC/db4_dest; \
        wget --quiet https://download.oracle.com/berkeley-db/$BDBVER.tar.gz; \
        echo "12edc0df75bf9abd7f82f821795bcee50f42cb2e5f76a6a281b85732798364ef  $BDBVER.tar.gz" | sha256sum -c - || exit 1; \
        tar -xzf $BDBVER.tar.gz; \
        cd $BDBVER/build_unix/; \
        sed s/__atomic_compare_exchange/__atomic_compare_exchange_db/g -i ../dbinc/atomic.h; \
        ../dist/configure --enable-cxx --disable-shared --with-pic --prefix=$SRC/db4_dest; \
        make install; \
        cd $SRC; \
        rm -rf $BDBVER $BDBVER.tar.gz db4_dest/docs

# Download Devcoind source
RUN git clone --depth=1 https://github.com/devcoin/core.git $SRC/devcoin
# Compile & Install Devcoind
RUN set -ex; \
        cd $SRC/devcoin; \
        ./autogen.sh; \
        ./configure  --enable-cxx --disable-shared --with-pic BDB_LIBS="-L$SRC/db4_dest/lib -ldb_cxx-4.8" BDB_CFLAGS="-I$SRC/db4_dest/include"; \
        make
RUN set -ex; mkdir /inst; cd $SRC/devcoin; make install-strip DESTDIR=/inst

FROM emfox/light-baseimage:1.3.4

COPY --from=builder /inst/usr/local /usr/local
# Update packages and install package dependencies
RUN apt-get -y update \
    && /container/tool/add-multiple-process-stack \
    && LC_ALL=C DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
           libevent-pthreads-2.1-7 libevent-2.1-7 libcurl4\
           libboost-system1.74.0 libboost-filesystem1.74.0 libboost-program-options1.74.0 libboost-thread1.74.0 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Create devcoin user
RUN useradd devcoin; chsh -s /bin/bash devcoin; mkdir -p /home/devcoin/.devcoin

# Configure devcoind settings
RUN echo '#Noob config file for devcoind, should be enough!\n\
server=1\n\
daemon=0\n\
rpcuser=devcoinrpc\n\
rpcpassword=a_s3cret_password\n\
addnode=LFM.Knotwork.com\n\
addnode=Server1.Knotwork.net\n\
rpcbind=127.0.0.1\n\
rpcbind=devcoin\n\
rpcallowip=127.0.0.1/24\n\
rpcallowip=172.16.0.0/12\n\
rpcport=52332\n\
# Set gen=1 to attempt to generate coins\n\
gen=0\n\
# Allow direct connections for the 'pay via IP address' feature.\n\
allowreceivebyip=1\n\
# Enable transaction index\n\
txindex=1\n\
wallet=wallet\n\
'\
> /home/devcoin/.devcoin/devcoin.conf
RUN chown devcoin: -R /home/devcoin

# Setup devcoind service
ENV DEVCOIND /container/run/process/devcoind
RUN mkdir /var/log/devcoind; mkdir -p $DEVCOIND/log; echo '#!/bin/sh\n\
\n\
exec 2>&1\n\
\n\
cd /home/devcoin/; exec /sbin/setuser devcoin devcoind\n\
'\
> $DEVCOIND/run
RUN echo '#!/bin/sh\n\
exec /usr/bin/svlogd -tt /var/log/devcoind\n\
'\
> $DEVCOIND/log/run
RUN chmod +x $DEVCOIND/run; chmod +x $DEVCOIND/log/run; ln -s $DEVCOIND /etc/service/

EXPOSE 52332
