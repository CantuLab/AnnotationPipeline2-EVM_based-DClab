[Home](README.md) | [← Training set creation](03-Training_set.md) | [Next: Ab initio prediction →](05-Ab_initio_prediction.md)

---

# 04 — Predictor training

Train Augustus and GeneMark-ET on the PASA-derived gene models from [step 03](03-Training_set.md). Both trained models are used in [step 05](05-Ab_initio_prediction.md) for genome-wide prediction.

---

## 4.1. Augustus training

```bash
cd "${AUGUSTUS_TRAIN_DIR}"
cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")

export PATH="${AUGUSTUS_DIR}/bin:${AUGUSTUS_DIR}:${PATH}"
export AUGUSTUS_CONFIG_PATH="${AUGUSTUS_CONFIG}"
```

### 4.1.1. Convert training models to GenBank format

```bash
"${AUGUSTUS_DIR}/scripts/gff2gbSmallDNA.pl" \
  "${TRAINING_DIR}/mRNA_training_PASA.sqlite.assemblies.fasta.transdecoder.genome.clean.gff3" \
  "${GENOME}" \
  300 \
  "${GENOME_PREFIX}.gene_models.augustus_genebank.gb"
```

### 4.1.2. Train Augustus

```bash
"${AUGUSTUS_DIR}/scripts/autoAugTrain.pl" \
  --verbose \
  --trainingset "${GENOME_PREFIX}.gene_models.augustus_genebank.gb" \
  --genome "${GENOME}" \
  --species "${GENOME_PREFIX}" \
  --optrounds 5 --CRF \
  --cpus=${THREADS} \
  > augustus.train.log 2> augustus.train.err
```

### 4.1.3. Back up the trained model

Augustus writes the model into its config directory. Save a local copy in case the config directory is shared:

```bash
rsync -av "${AUGUSTUS_CONFIG}/species/${GENOME_PREFIX}" .
```

---

## 4.2. GeneMark-ET training

> **License:** GeneMark-ET requires a free license key from the [GeneMark website](http://exon.gatech.edu/GeneMark/). Place it at `~/.gm_key` before running.

```bash
# One-time license key setup
zcat "${GM_KEY_FILE}" > gm_key && cp gm_key ~/.gm_key

cd "${GENEMARK_TRAIN_DIR}"
cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")

# GeneMark-ET uses intron hints from the PASA training set
awk '$3=="intron"' \
  "${TRAINING_DIR}/mRNA_training_PASA.sqlite.assemblies.fasta.transdecoder.genome.clean.gff3" \
  > "${GENOME_PREFIX}.gene_models.introns.gff3"
```

### 4.2.1. Run GeneMark training

```bash
"${GENEMARK_DIR}/gmes_petap.pl" \
  --sequence "${GENOME}" \
  --ET "${GENOME_PREFIX}.gene_models.introns.gff3" \
  --training \
  --cores ${THREADS} --v \
  --max_intergenic 1000000 \
  --max_intron 20000 \
  > genemark.train.log 2> genemark.train.err
```

### 4.2.2. Retrieve the trained model

```bash
rsync -L run/ET_C.mod ./ && mv ET_C.mod "${GENOME_PREFIX}.genemark.mod"
```

---

[Home](README.md) | [← Training set creation](03-Training_set.md) | [Next: Ab initio prediction →](05-Ab_initio_prediction.md)
