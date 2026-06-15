[Home](README.md) | [Next: External evidences →](01-External_evidences.md)

---

# 00 — Setup

## Prerequisites

### Software

| Tool | Tested version | Purpose |
|---|---|---|
| <a href="https://www.repeatmasker.org/" target="_blank">RepeatMasker</a> | 4.1.7-p1 | Repeat annotation |
| <a href="http://research-pub.gene.com/gmap/" target="_blank">GMAP</a> | 2024-11-20 | Transcript alignment |
| <a href="https://github.com/icebert/pblat" target="_blank">pblat</a> | 2.5.1 | Transcript alignment (fast BLAT) |
| <a href="https://github.com/PASApipeline/PASApipeline" target="_blank">PASA</a> | 2.5.3 | Transcript assembly and gene model polishing |
| <a href="https://github.com/Gaius-Augustus/Augustus" target="_blank">Augustus</a> | 3.5.0 | Ab initio gene prediction |
| <a href="http://exon.gatech.edu/GeneMark/" target="_blank">GeneMark-ET</a> | gmes 4 | Ab initio gene prediction (requires a free license key) |
| <a href="https://github.com/EVidenceModeler/EVidenceModeler" target="_blank">EVidenceModeler</a> | 2.1.0 | Gene model consensus |
| <a href="https://github.com/gpertea/gffread" target="_blank">gffread</a> | any | CDS/protein extraction from GFF3 |
| <a href="http://www.htslib.org/" target="_blank">samtools</a> | any | FASTA indexing |
| <a href="https://www.gnu.org/software/parallel/" target="_blank">GNU parallel</a> | any | Parallel execution |

### Custom scripts

The following lab scripts are used in steps 3, 7, 8, and 9. Replace them with equivalent tools if unavailable:

- `GFF_extract_features.py` — filters and validates GFF3 gene models (removes incomplete CDS, short proteins, multi-copy genes)
- `GFF_RenameThemAll.py` — applies a standardized gene nomenclature to a GFF3 file
- `getLengthFromFasta.py` — reports sequence length from a FASTA file

### Input data

- A genome assembly in FASTA format
- External transcript evidence in FASTA format: Iso-Seq HQ reads, CDS from closely related species, or RNA-seq assemblies
- A repeat library appropriate for your organism (FASTA format)

---

## Configuration

**Set these variables once at the start of your session.** All commands in subsequent steps reference them.

```bash
# ── Input files ──────────────────────────────────────────────────────────────
# Genome assembly FASTA
GENOME_FASTA="/path/to/genome.fasta"

# Short prefix used for all output file names (e.g. species abbreviation + assembly version)
GENOME_PREFIX="SpeciesName_v1.0"

# ── Output directory structure ────────────────────────────────────────────────
ANNOT_DIR="/path/to/annotation"          # root annotation directory
EXTERNAL_DIR="${ANNOT_DIR}/00-External_evidences/mRNAs"
REPEAT_DIR="${ANNOT_DIR}/01-Repeats"
TRAINING_DIR="${ANNOT_DIR}/02-Training_set"
AUGUSTUS_TRAIN_DIR="${ANNOT_DIR}/03-Predictor_training/Augustus"
GENEMARK_TRAIN_DIR="${ANNOT_DIR}/03-Predictor_training/Genemark"
AUGUSTUS_PRED_DIR="${ANNOT_DIR}/04-Prediction/Augustus"
GENEMARK_PRED_DIR="${ANNOT_DIR}/04-Prediction/Genemark"
EVM_DIR="${ANNOT_DIR}/05-EVM"
POLISHING_DIR="${ANNOT_DIR}/06-Polishing"
FILTERING_DIR="${ANNOT_DIR}/07-Filtering"

# ── Tool paths ────────────────────────────────────────────────────────────────
REPEATMASKER_DIR="/path/to/RepeatMasker"       # directory containing RepeatMasker binary
REPEAT_LIB="/path/to/repeat_library.lib"       # organism-specific repeat library (FASTA)
GMAP_DIR="/path/to/gmap"                       # gmap installation root
BLAT_DIR="/path/to/pblat"                      # pblat installation root
PASA_DIR="/path/to/PASApipeline"               # PASA installation root
AUGUSTUS_DIR="/path/to/Augustus"               # Augustus installation root
AUGUSTUS_CONFIG="${AUGUSTUS_DIR}/config"
GENEMARK_DIR="/path/to/gmes_linux_64"          # GeneMark-ET directory
EVM_TOOLS="/path/to/EVidenceModeler"           # EVM installation root
GFFREAD="/path/to/gffread"                     # gffread binary
FASTA_BIN="/path/to/fasta/bin"                 # fasta toolkit bin directory (used in polishing)
SCRIPTS_DIR="/path/to/custom_scripts"          # directory with GFF_extract_features.py etc.

# GeneMark license key archive (downloaded from the GeneMark website)
GM_KEY_FILE="/path/to/gm_key.gz"

# ── Run parameters ────────────────────────────────────────────────────────────
THREADS=48             # CPU threads for parallelized steps
MAX_INTRON=25000       # Maximum intron length (bp); increase for species with large introns
```

Create the output directories:

```bash
mkdir -p "${EXTERNAL_DIR}" "${REPEAT_DIR}" "${TRAINING_DIR}" \
         "${AUGUSTUS_TRAIN_DIR}" "${GENEMARK_TRAIN_DIR}" \
         "${AUGUSTUS_PRED_DIR}" "${GENEMARK_PRED_DIR}" \
         "${EVM_DIR}" "${POLISHING_DIR}" "${FILTERING_DIR}"
```

---

[Home](README.md) | [Next: External evidences →](01-External_evidences.md)
