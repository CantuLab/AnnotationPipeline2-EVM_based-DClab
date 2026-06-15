[Home](README.md) | [← Predictor training](04-Predictor_training.md) | [Next: Evidence alignment →](06-Evidence_alignment.md)

---

# 05 — Ab initio prediction

Run genome-wide gene prediction using the models trained in [step 04](04-Predictor_training.md). Three prediction tracks are produced and all passed to EVM in [step 07](07-EVM_consensus.md): Augustus, GeneMark, and PASA TransDecoder (the latter reused from [step 03](03-Training_set.md)).

---

## 4.1. Augustus prediction

```bash
cd "${AUGUSTUS_PRED_DIR}"
mkdir -p fasta gff3 logs

export PATH="${AUGUSTUS_DIR}/bin:${AUGUSTUS_DIR}:${PATH}"
export AUGUSTUS_CONFIG_PATH="${AUGUSTUS_CONFIG}"

cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")
```

### 4.1.1. Split genome into per-sequence FASTA files

```bash
awk 'BEGIN {getline; id=$1; gsub(">","",id); filename="fasta/"id".fasta"; print $0 > filename}
     { if ($1~"^>") {id=$1; sub(/>/,"",id); filename="fasta/"id".fasta"}; print $0 > filename }
     END {print "done"}' < "${GENOME}" \
  | sed 's:^:fasta/:;s:$:.fasta:' > fasta_list
```

### 4.1.2. Run Augustus in parallel on each sequence

```bash
parallel --gnu -j ${THREADS} \
  "augustus --species=${GENOME_PREFIX} --singlestrand=true --gff3=on \
   --outfile=gff3/{/.}.gff3 --noInFrameStop=true {} &> logs/{/.}.log" \
  :::: fasta_list
```

### 4.1.3. Merge results into a single GFF3

```bash
for file in gff3/*.gff3; do
  seq_name=$(basename "$file" .gff3)
  sed 's:sequence:Sequence:;s:seq:'"${seq_name}"':g;s:=:='"${seq_name}"'.:g;s:;Name=.*::' "$file"
done \
  | grep -v '^#' \
  | sed 's:transcript:mRNA:;/codon/d;/intron/d' \
  | sed '/CDS/ s:\(.*\)CDS\(.*\)ID=\(.*Parent=\)\(.*\):\1exon\2Parent=\4;ID=\4.exon\n\1CDS\2ID=\3\4:' \
  | awk 'BEGIN {count=1; OFS="\t"} $3=="exon" {$0=$0count; count+=1} {print $0}' \
  > prediction.augustus.gff3

# Extract CDS sequences for a quick sanity check
"${GFFREAD}" -x prediction.augustus.CDS.fasta -g "${GENOME}" prediction.augustus.gff3
```

### 4.1.4. Copy to EVM directory

```bash
cp prediction.augustus.gff3 "${EVM_DIR}/"
```

---

## 4.2. GeneMark prediction

```bash
cd "${GENEMARK_PRED_DIR}"
# Reuse the genome splits from the Augustus step
ln -s "${AUGUSTUS_PRED_DIR}/fasta" .
ln -s "${AUGUSTUS_PRED_DIR}/fasta_list" .
mkdir -p gff3 logs

cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")
GENEMARK_MODEL="${GENEMARK_TRAIN_DIR}/${GENOME_PREFIX}.genemark.mod"
```

### 4.2.1. Run GeneMark in parallel

```bash
parallel --gnu -j ${THREADS} \
  "${GENEMARK_DIR}/gmhmme3 -m ${GENEMARK_MODEL} -o gff3/{/.}.gff3 -b logs/{/.}.stats -f gff3 {}" \
  :::: fasta_list
```

### 4.2.2. Merge results into a single GFF3

```bash
for file in gff3/*.gff3; do
  seq_name=$(basename "$file" .gff3)
  sed 's:sequence:Sequence:;s:seq:'"${seq_name}"':g;s:=:='"${seq_name}"'.:g;s:;Name=.*::' "$file"
done \
  | grep -v '^#' \
  | sed 's:transcript:mRNA:;/codon/d;/Intron/d' \
  | sed '/CDS/ s:\(.*\)CDS\(.*\)ID=\(.*Parent=\)\(.*\):\1exon\2Parent=\4;ID=\4.exon\n\1CDS\2ID=\3\4:' \
  | awk 'BEGIN {count=1; OFS="\t"} $3=="exon" {$0=$0count; count+=1} {print $0}' \
  > prediction.genemark.gff3

"${GFFREAD}" -x prediction.genemark.CDS.fasta -g "${GENOME}" prediction.genemark.gff3
```

### 4.2.3. Copy to EVM directory

```bash
cp prediction.genemark.gff3 "${EVM_DIR}/"
```

---

## 4.3. PASA prediction

The PASA transcript-based gene models from [step 03](03-Training_set.md) are used as a third, high-confidence prediction track in EVM:

```bash
cd "${TRAINING_DIR}"
cat mRNA_training_PASA.sqlite.assemblies.fasta.transdecoder.genome.clean.gff3 \
  | sed 's:\~\~:-:g;s:transdecoder:PASA_transdecoder:;s:;Name=.*::' \
  > "${EVM_DIR}/prediction.pasa.gff3"
```

---

[Home](README.md) | [← Predictor training](04-Predictor_training.md) | [Next: Evidence alignment →](06-Evidence_alignment.md)
