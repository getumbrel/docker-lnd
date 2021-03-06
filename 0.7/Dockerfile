# This Dockerfile build lnd v0.7.x twice: once on Alpine, once on Debian
# If the binaries are the same, one is compressed, and copied to the `final` stage

# lnd version to be build
ARG VERSION=v0.7.1-beta

# Target CPU archtecture of built lnd binary
ARG ARCH

# Define default versions so that they don't have to be repreated throughout the file
ARG VER_GO=1.14
ARG VER_ALPINE=3.11

ARG USER=lnd
ARG DIR=/data/

#
## NOTE: You should only override the ARGs below, if you know what you're doing
#
ARG PKG=github.com/lightningnetwork/lnd

# original content
#   src: https://github.com/lightningnetwork/lnd/blob/v0.7.1-beta/release.sh#L63-L64
ARG TAGS_BASE="autopilotrpc invoicesrpc walletrpc routerrpc watchtowerrpc"
ARG TAGS_LND="signrpc chainrpc"

# Added to make output binary static
#   ctx: https://github.com/golang/go/issues/26492
ARG TAGS_STATIC="osusergo netgo static_build"

# Added by yours truly (@lncm)
ARG TAGS_EXTRA="wtclientrpc"

# Force Go to use the cgo based DNS resolver. This is required to ensure DNS
#   queries required to connect to linked containers succeed
ARG GODEBUG="netdns=cgo"


#
## This stage fetches and verifies source code, and applies necessary fixes
#
FROM golang:${VER_GO}-alpine${VER_ALPINE} AS preparer

ARG VERSION
ARG VER_GO

RUN apk add --no-cache  gnupg  git

ENV KEYBASE_USER=roasbeef
ENV KEYS F8037E70C12C7A263C032508CE58F7F8E20FD9A2

# First, try to import key currently on @roasbeef's keybase account into GPG,
# Second, also try to fetch that key from keyservers (in case it's not his key, or he already discarded it…).
#   This command doesn't stop the flow on error, and
#   Key verification happens in the next step
RUN wget -qO- "https://keybase.io/$KEYBASE_USER/pgp_keys.asc" | gpg --import && \
    for srv in  keyserver.ubuntu.com  hkp://p80.pool.sks-keyservers.net:80  ha.pool.sks-keyservers.net  keyserver.pgp.com  pgp.mit.edu; do \
        timeout 9s  gpg  --keyserver "$srv"  --recv-key $KEYS  >/dev/null 2<&1 && \
            { echo "OK:  $srv" && exit 0; } || \
            { echo "ERR: $srv fail=$?"; } ; \
    done

RUN gpg --list-keys && \
    gpg --list-keys $KEYS

RUN mkdir -p /go/src/

# Fetch lnd source code
RUN cd /go/src/ && \
    git clone  -b "$VERSION"  --depth=1  https://github.com/lightningnetwork/lnd .

WORKDIR /go/src/

## Verify that git tag contains a valid signature
#   NOTE: The fallback condition is a hack around @Roasbeef's "key hygiene".  A manual attempt at accepting expired keys
#       through git verify-tag; What can possibly go wrong? 😅
#       More: https://github.com/lightningnetwork/lnd/issues/3507#issuecomment-532414524
RUN git verify-tag "$VERSION" || \
    { git verify-tag --raw "$VERSION" 2>&1 | grep EXPKEYSIG && echo "Accepting valid signature with an expired key!"; }

# NOTE: context: https://github.com/lightningnetwork/lnd/issues/3506
RUN go mod edit \
    -go="$VER_GO" \
    -replace="github.com/btcsuite/btcwallet=github.com/btcsuite/btcwallet@v0.0.0-20190814023431-505acf51507f"

RUN go mod tidy

# Show all differences applied on top of upstream
RUN git diff



#
## This stage builds `lnd` & `lncli` in Alpine environment
#
FROM golang:${VER_GO}-alpine${VER_ALPINE} AS alpine-builder

# Provided by Docker by default
ARG TARGETVARIANT

# These two should only be set for cross-compilation
ARG GOARCH
ARG GOARM

# Capture ARGs defined globally
ARG PKG
ARG TAGS_BASE
ARG TAGS_LND
ARG TAGS_STATIC
ARG TAGS_EXTRA
ARG GODEBUG

# Aggregate all tags together
ENV TAGS="$TAGS_BASE $TAGS_STATIC $TAGS_EXTRA"

# Only set GOOS if GOARCH is set
ENV GOOS ${GOARCH:+linux}

# If GOARM is not set, but TARGETVARIANT is set - hardcode GOARM to 6
ENV GOARM ${GOARM:-${TARGETVARIANT:+6}}

ENV LDFLAGS "-s -w -buildid="

RUN apk add --no-cache  musl-dev  git  gcc

RUN mkdir -p /go/src/

COPY  --from=preparer /go/src/  /go/src/

WORKDIR /go/src/

RUN env && go version && go env

## Build lnd binaries
#   The first line gets hash of the last git-commit & second one prints it.
#   And here's all other flags explained:
#       `-x` [verbocity++] print all executed commands
#       `-v` [verbocity++] print names of compiled packages
#       `-mod=readonly` [reproducibility] do not change versions of used packages no matter what
#       `-trimpath` [reproducibility] make sure absolute paths are not included anywhere in the binary
#       `-tags` [reproducibility] tell Go to build a static binary, see more: https://github.com/golang/go/issues/26492
#       `-ldflags`
#           `-s` [size--] do not include symbol table and debug info
#           `-w` [size--] do not include DWARF symbol table
#           `-buildid` [reproducibility] while this should always be the same in our setup, clear it just-in-case
#           `-X` [info] is used to inject git-commit into the built binaries
#
#   NOTE: all of this has to happen in a single `RUN`, because it's impossible to set ENV var in Docker to
#       an output of an expression
RUN export GIT_TAG="$(git describe --abbrev=40)"; \
    echo "Building git tag: $GIT_TAG"; \
    go build  -x  -v  -trimpath  -mod=readonly  -tags="$TAGS $TAGS_LND" \
        -ldflags="$LDFLAGS -X $PKG/build.Commit=$GIT_TAG" \
        -o /go/bin/  "$PKG/cmd/lnd" && \
    go build  -x  -v  -trimpath  -mod=readonly  -tags="$TAGS" \
        -ldflags="$LDFLAGS -X $PKG/build.Commit=$GIT_TAG" \
        -o /go/bin/  "$PKG/cmd/lncli"



#
## This stage builds `lnd` & `lncli` in Debian environment
#
# NOTE: Comments that would be identical to Alpine stage skipped for brevity
FROM golang:${VER_GO}-buster AS debian-builder

ARG TARGETVARIANT
ARG GOARCH
ARG GOARM
ARG PKG
ARG TAGS_BASE
ARG TAGS_LND
ARG TAGS_STATIC
ARG TAGS_EXTRA
ARG GODEBUG

ENV TAGS="$TAGS_BASE $TAGS_STATIC $TAGS_EXTRA"
ENV GOOS ${GOARCH:+linux}
ENV GOARM ${GOARM:-${TARGETVARIANT:+6}}
ENV LDFLAGS "-s -w -buildid="

RUN apt-get update  &&  apt-get -y install  file  git

RUN mkdir -p /go/src/

COPY  --from=preparer /go/src/  /go/src/

WORKDIR /go/src/

RUN env && go version && go env

RUN export GIT_TAG="$(git describe --abbrev=40)"; \
    echo "Building git tag: $GIT_TAG"; \
    go build  -x  -v  -trimpath  -mod=readonly  -tags="$TAGS $TAGS_LND" \
        -ldflags="$LDFLAGS -X $PKG/build.Commit=$GIT_TAG" \
        -o /go/bin/  "$PKG/cmd/lnd" && \
    go build  -x  -v  -trimpath  -mod=readonly  -tags="$TAGS" \
        -ldflags="$LDFLAGS -X $PKG/build.Commit=$GIT_TAG" \
        -o /go/bin/  "$PKG/cmd/lncli"



#
## This stage compares previously built binaries, and only proceeds if they are identical
#
FROM alpine:${VER_ALPINE} AS cross-check

# Install utilities used later
RUN apk add --no-cache  ca-certificates  file  upx

RUN mkdir -p  /bin  /alpine  /debian

# Copy binaries from build stages
COPY  --from=alpine-builder /go/bin/*  /alpine/
COPY  --from=debian-builder /go/bin/*  /debian/

# Print binary info PRIOR comparison & compression
RUN sha256sum   /debian/*  /alpine/*
RUN file        /debian/*  /alpine/*
RUN du          /debian/*  /alpine/*

# Compare both built binaries
RUN diff -q  /alpine/lnd   /debian/lnd \
 && diff -q  /alpine/lncli /debian/lncli

# If identical, proceed to move the binaries into `/bin/`
RUN mv  /alpine/*  /bin/

# Compress, and be verbose about it
RUN upx -v /bin/lnd  /bin/lncli

# Print binaries info PAST compression
RUN sha256sum   /bin/lnd  /bin/lncli
RUN file        /bin/lnd  /bin/lncli
RUN du          /bin/lnd  /bin/lncli



#
## This stage is used to generate /etc/{group,passwd,shadow} files & avoid RUN-ing commands in the `final` layer,
#   which would break cross-compiled images.
#
FROM alpine:${VER_ALPINE} AS perms

ARG USER
ARG DIR

# NOTE: Default GID == UID == 1000
RUN adduser --disabled-password \
            --home "$DIR" \
            --gecos "" \
            "$USER"

# Needed to prevent `VOLUME ${DIR}/.lnd/` creating it with `root` as owner
USER $USER
RUN mkdir -p "$DIR/.lnd/"



#
## This is the final image that gets shipped to Docker Hub
#
# NOTE: `${ARCH:+${ARCH}/}` - if ARCH is set, append `/` to it, leave it empty otherwise
FROM ${ARCH:+${ARCH}/}alpine:${VER_ALPINE} AS final

ARG USER
ARG DIR

LABEL maintainer="Damian Mee (@meeDamian)"

# Copy only the relevant parts from the `perms` image
COPY  --from=perms /etc/group /etc/passwd /etc/shadow  /etc/

# From `perms`, copy *the contents* of `${DIR}` (ie. `.lnd/`), and set correct owner for destination `${DIR}`
COPY  --from=perms --chown=$USER:$USER $DIR  $DIR

#
COPY  --from=cross-check /etc/ssl/certs/ca-certificates.crt  /etc/ssl/certs/

# Copy binaries from the cross-check stage
COPY  --from=cross-check /bin/lnd /bin/lncli  /usr/local/bin/

USER $USER

# Expose volume containing all `lnd` data
VOLUME $DIR/.lnd/

# Expose lnd ports (rest, p2p, rpc respectively)
EXPOSE  8080  9735  10009

# Specify the start command and entrypoint as the lnd daemon
ENTRYPOINT ["lnd"]
