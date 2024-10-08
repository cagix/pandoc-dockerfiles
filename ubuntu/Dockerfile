# Base ##################################################################
ARG base_image_version=noble
FROM ubuntu:$base_image_version AS ubuntu-builder-base
WORKDIR /app

## An ARG instruction goes out of scope at the end of the build stage 
## where it was defined. To use an argument in multiple stages, each stage 
## must include the ARG instruction
ARG base_image_version=noble
ARG lua_version=5.4
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get -q --no-allow-insecure-repositories update \
  && apt-get install --assume-yes --no-install-recommends \
       build-essential=* \
       ca-certificates=* \
       cabal-install=* \
       curl=* \
       fakeroot=* \
       git \
       ghc=* \
       libgmp-dev=2:6.* \
       liblua$lua_version-dev=* \
       pkg-config=* \
       zlib1g-dev=* \
  && rm -rf /var/lib/apt/lists/*

COPY cabal.root.config /root/.cabal/config
RUN cabal --version \
  && ghc --version \
  && cabal update

# Builder ###############################################################
FROM ubuntu-builder-base AS ubuntu-builder
ARG pandoc_commit=main
RUN git clone --branch=$pandoc_commit --depth=1 --quiet \
  https://github.com/jgm/pandoc /usr/src/pandoc

# Remove the settings that ship with pandoc, unless we're building the main
# branch.
RUN [ "${pandoc_commit}" = "main" ] || rm -f cabal.project

COPY ./ubuntu/freeze/pandoc-$pandoc_commit.project.freeze \
     /usr/src/pandoc/cabal.project.freeze

# Install Haskell dependencies
WORKDIR /usr/src/pandoc
# Add pandoc-crossref to project
ARG without_crossref=
ARG extra_packages="pandoc-cli pandoc-crossref"
RUN test -n "$without_crossref" || \
    printf "extra-packages: pandoc-crossref\n" > cabal.project.local;

# Build pandoc and pandoc-crossref. The `allow-newer` is required for
# when pandoc-crossref has not been updated yet, but we want to build
# anyway.
RUN cabal update \
  && cabal build \
      --allow-newer 'lib:pandoc' \
      --disable-tests \
      --disable-bench \
      --jobs \
      . $extra_packages

# Cabal's exec stripping doesn't seem to work reliably, let's do it here.
RUN find dist-newstyle \
         -name 'pandoc*' -type f -perm -u+x \
         -exec strip '{}' ';' \
         -exec cp '{}' /usr/local/bin/ ';'

# Minimal ###############################################################
FROM ubuntu:$base_image_version AS ubuntu-minimal
ARG pandoc_version=edge
ARG lua_version=5.4
LABEL maintainer='Albert Krewinkel <albert+pandoc@tarleb.com>'
LABEL org.pandoc.maintainer='Albert Krewinkel <albert+pandoc@tarleb.com>'
LABEL org.pandoc.author "John MacFarlane"
LABEL org.pandoc.version "$pandoc_version"

# Set and create user data directory
ENV XDG_DATA_HOME=/usr/local/share
WORKDIR $XDG_DATA_HOME/pandoc

# Set default working directory
WORKDIR /data
ENTRYPOINT ["/usr/local/bin/pandoc"]

COPY --from=ubuntu-builder \
  /usr/local/bin/pandoc \
  /usr/local/bin/

# Add pandoc symlinks and install runtime dependencies
RUN ln -s /usr/local/bin/pandoc /usr/local/bin/pandoc-lua \
  && ln -s /usr/local/bin/pandoc /usr/local/bin/pandoc-server \
  && apt-get -q --no-allow-insecure-repositories update \
  && DEBIAN_FRONTEND=noninteractive \
     apt-get install --assume-yes --no-install-recommends \
       ca-certificates=\* \
       liblua$lua_version-0=\* \
       libatomic1=\* \
       libgmp10=\* \
       libpcre3=\* \
       libyaml-0-2=\* \
       zlib1g=\* \
  && rm -rf /var/lib/apt/lists/*

# Core ##################################################################
FROM ubuntu-minimal AS ubuntu-core
COPY --from=ubuntu-builder \
  /usr/local/bin/pandoc-crossref \
  /usr/local/bin/

# Additional packages frequently used during conversions
# NOTE: `libsrvg`, pandoc uses `rsvg-convert` for working with svg images.
RUN apt-get -q --no-allow-insecure-repositories update \
  && DEBIAN_FRONTEND=noninteractive \
     apt-get install --assume-yes --no-install-recommends \
       librsvg2-bin=2.* \
  && rm -rf /var/lib/apt/lists/*

# LaTeX ##############################################################
FROM ubuntu-core AS ubuntu-latex

# NOTE: to maintainers, please keep this listing alphabetical.
RUN apt-get -q --no-allow-insecure-repositories update \
  && DEBIAN_FRONTEND=noninteractive \
     apt-get install --assume-yes --no-install-recommends \
        fontconfig \
        gnupg \
        gzip \
        libfontconfig1 \
        libfreetype6 \
        perl \
        tar \
        wget \
        xzdec \
  && rm -rf /var/lib/apt/lists/*

# TeXLive binaries location
ARG texlive_bin="/opt/texlive/texdir/bin"

RUN TEXLIVE_ARCH="$(uname -m)-$(uname -s | tr '[:upper:]' '[:lower:]')" && \
    mkdir -p ${texlive_bin} && \
    ln -sf "${texlive_bin}/${TEXLIVE_ARCH}" "${texlive_bin}/default"

# Modify PATH environment variable, prepending TexLive bin directory
ENV PATH="${texlive_bin}/default:${PATH}"

WORKDIR /root

COPY common/latex/texlive.profile /root/texlive.profile
COPY common/latex/install-texlive.sh /root/install-texlive.sh
COPY common/latex/packages.txt /root/packages.txt

# TeXLive version to install (leave empty to use the latest version).
ARG texlive_version=

# TeXLive mirror URL (leave empty to use the default mirror).
ARG texlive_mirror_url=

RUN ( [ -z "$texlive_version"    ] || printf '-t\n%s\n"' "$texlive_version" \
    ; [ -z "$texlive_mirror_url" ] || printf '-m\n%s\n' "$texlive_mirror_url" \
    ) | xargs /root/install-texlive.sh \
  && sed -e 's/ *#.*$//' -e '/^ *$/d' /root/packages.txt | \
     xargs tlmgr install \
  && rm -f /root/texlive.profile \
           /root/install-texlive.sh \
           /root/packages.txt \
  && TERM=dumb luaotfload-tool --update \
  && chmod -R o+w /opt/texlive/texdir/texmf-var

WORKDIR /data

# extra ##############################################################
FROM ubuntu-latex AS ubuntu-extra

COPY common/extra/packages.txt /root/extra_packages.txt
COPY common/extra/requirements.txt /root/extra_requirements.txt

# The option `--break-system-packages` sounds bad but this is not
# really a problem here because we are not using Python debian packages
# anyway.
RUN apt-get -q --no-allow-insecure-repositories update \
  && apt-get install --assume-yes --no-install-recommends python3-pip \
  && pip3 install -r /root/extra_requirements.txt --break-system-packages \
  && rm -f /root/extra_requirements.txt

RUN sed -e 's/ *#.*$//' -e '/^ *$/d' /root/extra_packages.txt | \
    xargs tlmgr install \
  && rm -f /root/extra_packages.txt

# Templates
#
ARG PANDOC_DATA_DIR=$XDG_DATA_HOME/pandoc
ARG TEMPLATES_DIR=$PANDOC_DATA_DIR/templates
RUN mkdir -p $TEMPLATES_DIR

# eisvogel
ARG EISVOGEL_REPO=https://raw.githubusercontent.com/Wandmalfarbe/pandoc-latex-template
ARG EISVOGEL_VERSION=2.4.2
RUN wget ${EISVOGEL_REPO}/${EISVOGEL_VERSION}/eisvogel.tex \
        -O ${TEMPLATES_DIR}/eisvogel.latex

# Lua Filters
ARG LUA_FILTERS_REPO=https://github.com/pandoc/lua-filters/releases/download
ARG LUA_FILTERS_VERSION=v2021-11-05
RUN wget ${LUA_FILTERS_REPO}/${LUA_FILTERS_VERSION}/lua-filters.tar.gz \
    && tar xzf lua-filters.tar.gz \
        --strip-components=1 \
        --one-top-level=$PANDOC_DATA_DIR \
    && rm -f lua-filters.tar.gz

# tectonic
ARG TARGETARCH
ARG TECTONIC_REPO=https://github.com/tectonic-typesetting/tectonic/releases/download
ARG TECTONIC_VERSION=0.15.0
RUN if [ "$TARGETARCH" = "amd64" ] ; then TECTONIC_ARCH='x86_64'  ; \
    elif [ "$TARGETARCH" = "arm64" ] ; then TECTONIC_ARCH='aarch64' ; \
    else echo 'unsupported target arch for tectonic'; exit 1 ; \
    fi \
    && TECTONIC_TARBALL=tectonic-${TECTONIC_VERSION}-${TECTONIC_ARCH}-unknown-linux-gnu.tar.gz \
    && wget ${TECTONIC_REPO}/tectonic%40${TECTONIC_VERSION}/${TECTONIC_TARBALL} \
    && tar xzf ${TECTONIC_TARBALL} -C /usr/local/bin/ \
    && rm -f ${TECTONIC_TARBALL}
