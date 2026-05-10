# =============================================================================
# Dockerfile — trackplot ChIP-seq Visualization Environment
# =============================================================================
# Builds a reproducible environment containing:
#   - R 4.3 + required R packages (trackplot, data.table, tidyverse, ggplot2)
#   - bwtool (compiled from source)
#   - MySQL client (required by trackplot for UCSC genome browser queries)
#   - Quarto CLI (for rendering .qmd documents)
#
# Build:
#   docker build -t trackplot-env .
#
# Run (interactive R session):
#   docker run --rm -it -v $(pwd):/workspace trackplot-env R
#
# Run (render Quarto document):
#   docker run --rm -v $(pwd):/workspace trackplot-env \
#     quarto render /workspace/trackplot_chipseq.qmd
#
# Run (RStudio Server on http://localhost:8787, user: rstudio, pass: rstudio):
#   docker run --rm -p 8787:8787 -e PASSWORD=rstudio \
#     -v $(pwd):/workspace trackplot-env
# =============================================================================

FROM rocker/rstudio:4.3.2

LABEL maintainer="Danya Hassan"
LABEL description="trackplot ChIP-seq visualization environment"

# =============================================================================
# System dependencies
# =============================================================================
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Build tools for compiling bwtool and its dependencies
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    git \
    wget \
    curl \
    # MySQL client (required by trackplot for UCSC browser queries)
    default-mysql-client \
    libmysqlclient-dev \
    # R package build dependencies
    libcurl4-openssl-dev \
    libssl-dev \
    libxml2-dev \
    libfontconfig1-dev \
    libharfbuzz-dev \
    libfribidi-dev \
    libfreetype6-dev \
    libpng-dev \
    libtiff5-dev \
    libjpeg-dev \
    # Quarto dependency
    pandoc \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# =============================================================================
# Compile bwtool from source
# =============================================================================
# bwtool requires libbeato and zlib to be compiled first as dependencies.
# The specific libbeato commit is pinned for reproducibility.

WORKDIR /opt/build

RUN git clone https://github.com/CRG-Barcelona/libbeato.git && \
    git clone https://github.com/madler/zlib.git && \
    git clone https://github.com/CRG-Barcelona/bwtool.git && \
    # Compile libbeato at pinned commit
    cd libbeato && \
    git checkout 0c30432af9c7e1e09ba065ad3b2bc042baa54dc2 && \
    ./configure && make && \
    # Compile zlib
    cd ../zlib && \
    ./configure && make && \
    # Compile bwtool linking against libbeato and zlib
    cd ../bwtool && \
    ./configure \
        CFLAGS='-I../libbeato -I../zlib' \
        LDFLAGS='-L../libbeato/jkweb -L../libbeato/beato -L../zlib' && \
    make && \
    # Install bwtool to system PATH
    cp bwtool /usr/local/bin/bwtool && \
    chmod +x /usr/local/bin/bwtool && \
    # Verify installation
    bwtool 2>&1 | head -3

# =============================================================================
# Install Quarto CLI
# =============================================================================
RUN wget -q https://github.com/quarto-dev/quarto-cli/releases/download/v1.4.550/quarto-1.4.550-linux-amd64.deb \
    -O /tmp/quarto.deb && \
    dpkg -i /tmp/quarto.deb && \
    rm /tmp/quarto.deb

# =============================================================================
# Install R packages
# =============================================================================
RUN R -e "\
    install.packages(c( \
        'data.table', \
        'ggplot2', \
        'dplyr', \
        'tidyr', \
        'tidyverse', \
        'remotes', \
        'BiocManager' \
    ), repos = 'https://cloud.r-project.org', quiet = TRUE); \
    # Install trackplot from GitHub \
    remotes::install_github('poisonalien/trackplot', quiet = TRUE); \
    "

# =============================================================================
# Environment
# =============================================================================
ENV PATH="/usr/local/bin:${PATH}"
WORKDIR /workspace

# Default command: launch R
CMD ["R"]
