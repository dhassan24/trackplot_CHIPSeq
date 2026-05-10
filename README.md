# ChIP-seq Visualization with trackplot

IGV-style locus track plots and genome-wide metagene profiles from ENCODE
ChIP-seq data using the [trackplot](https://github.com/PoisonAlien/trackplot)
R package, applied to H1 hESC data for OCT4, NANOG, and histone marks.

## Workflow

```
ENCODE BigWig + NarrowPeak Files
        ↓
read_coldata() — link files to genome assembly
        ↓
track_extract() — extract signal at locus or gene
        ↓
track_plot() — IGV-style locus track plot
        (+ ChromHMM tracks, peak overlays, ideogram)
        ↓
profile_extract() + profile_summarize()
        ↓
profile_plot() — metagene TSS profile
```

## Repository Contents

| File | Description |
|------|-------------|
| `trackplot_chipseq.qmd` | Main analysis notebook (Quarto) |
| `Dockerfile` | Reproducible Docker image definition |
| `docker-compose.yml` | Compose services: RStudio, Quarto renderer, interactive R |
| `environment.yml` | Conda environment definition |

## Quick Start

### Option A — Docker Compose (recommended, fully reproducible)

```bash
# Build the image
docker compose build

# Start RStudio Server at http://localhost:8787
# username: rstudio  |  password: rstudio
docker compose up rstudio

# Render the Quarto document to HTML (runs and exits)
docker compose run --rm render

# Launch an interactive R session in the terminal
docker compose run --rm r

# Stop and clean up
docker compose down
```

### Option A (without Compose) — Plain Docker

```bash
# Build the image
docker build -t trackplot-env .

# Run RStudio Server at http://localhost:8787
docker run --rm -p 8787:8787 \
  -e PASSWORD=rstudio \
  -v $(pwd):/workspace \
  trackplot-env

# Render the Quarto document directly
docker run --rm \
  -v $(pwd):/workspace \
  trackplot-env \
  quarto render /workspace/trackplot_chipseq.qmd
```

### Option B — Conda

```bash
# Create and activate environment
conda env create -f environment.yml
conda activate trackplot-env

# Install trackplot R package from GitHub
Rscript -e "remotes::install_github('poisonalien/trackplot')"

# Compile and install bwtool (see below)
```

## Dependencies

| Tool | Purpose | Install |
|------|---------|---------|
| [trackplot](https://github.com/PoisonAlien/trackplot) | Track and profile plots | `remotes::install_github("poisonalien/trackplot")` |
| [bwtool](https://github.com/CRG-Barcelona/bwtool) | BigWig signal extraction | Compiled from source (see below) |
| MySQL client | UCSC genome browser queries | `conda install -c anaconda mysql` |
| Quarto | Render `.qmd` documents | [quarto.org](https://quarto.org) |
| data.table | Fast data manipulation | `install.packages("data.table")` |

## bwtool Installation

bwtool must be compiled from source. The Dockerfile handles this automatically.
For manual installation:

```bash
git clone https://github.com/CRG-Barcelona/libbeato.git
git clone https://github.com/madler/zlib.git
git clone https://github.com/CRG-Barcelona/bwtool.git

# Compile libbeato at pinned commit (for reproducibility)
cd libbeato
git checkout 0c30432af9c7e1e09ba065ad3b2bc042baa54dc2
./configure && make

# Compile zlib
cd ../zlib
./configure && make

# Compile bwtool
cd ../bwtool
./configure \
  CFLAGS='-I../libbeato -I../zlib' \
  LDFLAGS='-L../libbeato/jkweb -L../libbeato/beato -L../zlib'
make

# Install to user PATH
mkdir -p ~/bin
cp bwtool ~/bin/
chmod +x ~/bin/bwtool
export PATH=$HOME/bin:$PATH   # add to ~/.bashrc for persistence
```

If bwtool is not found inside an R session:

```r
Sys.setenv(PATH = paste(Sys.getenv("PATH"), "/home/<user>/bin", sep = ":"))
system("which bwtool")   # should return the path
```

## Data

This notebook uses publicly available ENCODE H1 hESC ChIP-seq data.
Download bigWig and narrowPeak files from the ENCODE portal:

| File | Description | ENCODE ID |
|------|-------------|-----------|
| `ENCFF465CMT.bigWig` | OCT4 ChIP-seq replicate 1 | ENCFF465CMT |
| `ENCFF996VEE.bigWig` | OCT4 ChIP-seq replicate 2 | ENCFF996VEE |
| `H1_Oct4.bed` | OCT4 narrowPeak calls | — |
| `H1_Nanog.bed` | NANOG narrowPeak calls | — |
| `H1_Ctcf.bed` | CTCF narrowPeak calls | — |
| `H1_k27ac.bed` | H3K27ac narrowPeak calls | — |
| `H1_k4me1.bed` | H3K4me1 narrowPeak calls | — |
| `H1_k4me3.bed` | H3K4me3 narrowPeak calls | — |
| `H1_H2az.bed` | H2A.Z narrowPeak calls | — |
| `H1_Pol2.bed` | RNA Pol II narrowPeak calls | — |

Place all files in a single directory and update `WORKDIR` in the
configuration chunk of the notebook.

## Notebook Sections

| Section | Description |
|---------|-------------|
| §5 — Locus track plots | IGV-style signal tracks at OCT4/POU5F1 locus |
| §5.5 — Overlay mode | Collapse multiple tracks into one for direct comparison |
| §5.6 — Multi-gene view | Display multiple genes with auto-scaled y-axes |
| §6 — NarrowPeak tracks | Peak BED files + ChromHMM chromatin state tracks |
| §7 — Metagene profiles | Genome-wide average signal around RefSeq TSS |
| §8 — Generic template | Drop-in template for any bigWig dataset |
| §9 — Troubleshooting | Common errors and fixes |

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `bwtool not found` | `Sys.setenv(PATH = paste(Sys.getenv("PATH"), "~/bin", sep=":"))` |
| MySQL errors | `conda install -c anaconda mysql` then restart R session |
| UCSC query timeouts | UCSC rate-limits requests; retry or download annotation locally |
| ChromHMM tracks not found | Check track names at https://genome.ucsc.edu/cgi-bin/hgTrackDb |
| `file not found` in read_coldata | Confirm `getwd()` matches where your bigWig files live |

## Notes

- The libbeato git commit (`0c30432`) is pinned in both the Dockerfile and
  manual instructions to ensure reproducible compilation of bwtool across
  environments.
- ChromHMM tracks are fetched live from UCSC — an internet connection is
  required for those cells unless annotation is downloaded locally.
- trackplot supports hg19, hg38, mm9, mm10, and any assembly with a UCSC
  Genome Browser presence. Change the `build` parameter in `read_coldata()`
  to switch assemblies.
