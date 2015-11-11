FROM alpine:latest

ENV GHC_VERSION      7.10.2
ENV PATCHELF_VERSION 0.8
ENV CABAL_VERSION    1.22.6.0

WORKDIR /tmp
COPY get-last-layer.sh build.mk ./
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
    curl -L https://www.haskell.org/ghc/dist/$GHC_VERSION/ghc-$GHC_VERSION-src.tar.bz2 | tar xj && \
    cd ghc-$GHC_VERSION && \
    cp -v /tmp/build.mk mk/build.mk && \
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
    cabal update && \
    cabal install --global stack && \
    stack setup && \
    : Clean up to keep the image small && \
    rm -rf /root/.cabal

WORKDIR /
RUN : "Layer 4: install extra packages" && \
    apk add mariadb-dev taglib-dev
COPY stack.yaml /root/.stack/global/stack.yaml
WORKDIR /
RUN : "Layer 5: install extra stack packages" && \
    stack install aeson aeson-pretty alex appar async attoparsec attoparsec-binary attoparsec-conduit auto-update base64-bytestring base-orphans blaze-builder blaze-html blaze-svg blaze-textual BoundedChan bytestring case-insensitive cereal clock colour conduit conduit-combinators conduit-extra containers criterion cryptonite data-default deepseq deepseq-generics directory either exceptions fast-logger filepath free happy hashable haskell-lexer haskell-src haskell-src-exts haskell-src-meta hex hslogger HsOpenSSL HTTP http-client http-client-tls http-conduit http-streams http-types HUnit quickcheck-instances interpolatedstring-perl6 language-haskell-extract lens lens-aeson lifted-async lifted-base monad-control monad-logger mono-traversable mtl network optparse-applicative parallel parsec parsers path-pieces pcre-heavy pcre-light persistent pipes pipes-bytestring primitive process profunctors QuickCheck random regex-base regex-compat regex-pcre regex-pcre-builtin regex-posix resourcet retry safe semigroupoids semigroups split stm stm-chans syb tagged tasty tasty-hunit tasty-quickcheck tasty-th test-framework test-framework-hunit test-framework-quickcheck2 test-framework-th text time timezone-olson timezone-series transformers transformers-base uniplate unix unordered-containers vector vector-algorithms vector-instances vector-th-unbox wai wai-extra warp yesod yesod-core yesod-static
COPY entrypoint.sh ./
ENTRYPOINT ["/entrypoint.sh"]
CMD ["bash"]
