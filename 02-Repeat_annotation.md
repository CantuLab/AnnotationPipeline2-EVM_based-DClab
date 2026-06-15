[Home](README.md) | [← External evidences](01-External_evidences.md) | [Next: Training set creation →](03-Training_set.md)

---

# 02 — Repeat annotation

RepeatMasker identifies and soft-masks repetitive elements in the genome (repetitive regions are converted to lowercase). Soft-masking is required by downstream gene predictors to avoid predicting genes within repeats.

The repeat GFF3 produced here is also passed directly to EVM in [step 07](07-EVM_consensus.md).

### 2.1. Run RepeatMasker

```bash
cd "${REPEAT_DIR}"
cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")

nohup "${REPEATMASKER_DIR}/RepeatMasker" \
  -pa 6 -s \
  -lib "${REPEAT_LIB}" \
  -a -lcambig -xsmall -poly -source -html -ace -gff -u -xm -excln -e ncbi \
  "${GENOME}" &
```

### 2.2. Convert RepeatMasker output to GFF3

```bash
"${REPEATMASKER_DIR}/util/rmOutToGFF3.pl" "${GENOME}.out" > "${GENOME}.out.gff3"
```

### 2.3. Copy to EVM directory (strip comment lines)

```bash
grep -v '^#' "${GENOME}.out.gff3" > "${EVM_DIR}/repeats.gff3"
```

### 2.4. Build repeat summary statistics

```bash
nohup "${REPEATMASKER_DIR}/util/buildSummary.pl" "${GENOME}.out" > "${GENOME}.out.stats" &
```

### 2.5. Create publication-ready repeat GFF3

```bash
{ echo "##gff-version 3"; sed '1,3d' "${GENOME}.out" \
  | awk '{OFS="\t"; print $5,"RepeatMasker","repeat",$6,$7,".",$9,".","ID="$15";match_ID="$10";class="$11}' \
  | awk '$7=="C"{$7="-"} 1' OFS="\t"; } \
  > "${GENOME_PREFIX}.repeats.gff3"
```

---

[Home](README.md) | [← External evidences](01-External_evidences.md) | [Next: Training set creation →](03-Training_set.md)
