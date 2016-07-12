FROM alpine:latest

ENV GHC_VERSION      7.10.3
ENV GHC_VERSION_S      7.10.3b
ENV PATCHELF_VERSION 0.8
ENV CABAL_VERSION    1.22.9.0

WORKDIR /tmp
COPY get-last-layer.sh build.mk ghc-linker-relocation.patch ./

COPY groupadd /usr/sbin/groupadd
COPY useradd /usr/sbin/useradd
COPY usermod /usr/sbin/usermod
COPY groupmod /usr/sbin/groupmod

#RUN apk --update add bash && \
#    /usr/sbin/groupadd -o --gid 1000 stack && \
#    /usr/sbin/useradd -oN --uid 1000 --gid 1000 --home /tmp stack

RUN : "Layer 1: fully working basic GHC in /usr/local" && \
    apk --update add curl xz alpine-sdk perl gmp-dev file gmp openssh openssl zlib-dev strace vim less jq ncurses-dev bash autoconf pcre-dev && \
    : Install patchelf && \
    curl -L https://nixos.org/releases/patchelf/patchelf-$PATCHELF_VERSION/patchelf-$PATCHELF_VERSION.tar.bz2 | tar xj && \
    cd patchelf-$PATCHELF_VERSION && \
    ./configure && \
    make install && \
    : Overlay ghc cross compiler image && \
    cd /tmp && \
    sh get-last-layer.sh nilcons/ghc-musl-auto ghc-cross | tar xzO | tar xJ -C / && \
    : Compile the ghc binary distribution && \
    cd /tmp && \
    curl -L https://downloads.haskell.org/~ghc/$GHC_VERSION/ghc-$GHC_VERSION_S-src.tar.bz2 | tar xj && \
    cd ghc-$GHC_VERSION && \
    cp -v /tmp/build.mk mk/build.mk && \
    patch -p1 < /tmp/ghc-linker-relocation.patch && \
    PATH=/opt/ghc-cross/bin:$PATH ./configure && \
    : patch libffi bug && \
    sed -i 's,chmod,sed -i s/__gnu_linux__/1/ libffi/build/src/closures.c \&\& chmod,' libffi/ghc.mk && \
    make -j8 && \
    make binary-dist && \
    : Install the ghc binary distribution to minimize docker layer size && \
    cd /tmp && \
    tar xjf ghc-$GHC_VERSION/ghc-$GHC_VERSION-x86_64-unknown-linux.tar.bz2 --transform "s/^ghc-${GHC_VERSION}/ghc-binary-dist/" && \
    cd ghc-binary-dist && \
    ./configure && \
    : musl ld requires --no-pie to work for some reason with ghc && \
    sed -i '/C\ compiler\ link/{ s/""/"--no-pie"/ }' settings && \
    make install && \
    : Clean up to keep the image small && \
    rm -rf /tmp/*


ENV PATH=/root/.cabal/bin:$PATH
RUN : "Layer 2: cabal-install, but only the binary, no executables" && \
    curl -L https://hackage.haskell.org/package/cabal-install-$CABAL_VERSION/cabal-install-$CABAL_VERSION.tar.gz | tar xz && \
    cd cabal-install-$CABAL_VERSION && \
    sed -i 's/export TMPDIR=.*/export TMPDIR=$(mktemp -t cabal-XXXXXX -d)/' bootstrap.sh && \
    EXTRA_CONFIGURE_OPTS=--disable-library-profiling ./bootstrap.sh && \
    : Clean up to keep the image small && \
    rm -rf /tmp/* /root/.ghc /root/.cabal/{lib,share}

WORKDIR /root/.cabal
RUN : "Layer 3: install stack" && \
    apk --update add sqlite-dev && \
    cabal update && \
    cabal install --global stack && \
    stack setup && \
    : Clean up to keep the image small && \
    rm -rf /root/.cabal

WORKDIR /
RUN : "Layer 4: install extra packages" && \
    apk add taglib-dev curl-dev icu-dev icu taglib icu-libs python python-dev
COPY stack.yaml /root/.stack/global/stack.yaml
WORKDIR /
RUN : "Layer 5: install extra stack packages" && \
    stack install aeson
CMD ["bash"]
