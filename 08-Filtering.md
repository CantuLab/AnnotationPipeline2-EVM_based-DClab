[Home](README.md) | [← EVM consensus & polishing](07-EVM_consensus.md) | [Next: Rename →](09-Renaming.md)

---

# 08 — Filtering

Remove gene models that are incomplete or of poor quality: proteins missing a stop codon and proteins shorter than 50 amino acids.

```bash
cd "${FILTERING_DIR}"
GENOME=$(basename "${GENOME_FASTA}")
```

### 8.1. Convert protein FASTA to tab-delimited format

Collapses multi-line FASTA sequences onto a single line per protein (ID in column 1, sequence in column 2), enabling reliable per-protein `grep` operations:

```bash
awk '/^>/{if(NR>1)printf("\n%s\t", substr($0,2)); else printf("%s\t", substr($0,2)); next} \
     {printf "%s",$0} END{print ""}' gene_models.protein.fasta \
  > gene_models.protein.fasta.tab
```

### 8.2. Identify proteins without a stop codon

In the protein FASTA, `*` marks the stop codon. Proteins lacking it have an incomplete CDS:

```bash
grep -v "\*" gene_models.protein.fasta.tab | cut -f1 \
  > gene_models.protein_IDs_without_stop_codon.txt
```

### 8.3. Identify short proteins (< 50 AA)

```bash
python2 "${SCRIPTS_DIR}/getLengthFromFasta.py" gene_models.protein.fasta \
  > gene_models.protein.fasta.len.txt
# Inspect this file and collect IDs with length < 50 AA
```

### 8.4. Build the removal list and filter

Combine the IDs from steps 8.2 and 8.3 into `gene_models.protein_IDs_to_remove.txt`, then filter:

```bash
grep -vf gene_models.protein_IDs_to_remove.txt gene_models.gff3 > gene_models_checked.gff3
```

Verify the filtered models are structurally valid:

```bash
cp "${GENOME_FASTA}" .

python2 "${SCRIPTS_DIR}/GFF_extract_features.py" \
  -a gene_models_checked.gff3 \
  -g "${GENOME}" \
  -p gene_models_checked_test \
  -lcmin \
  > gene_models_checked_test.log 2> gene_models_checked_test.err

awk '$3=="gene"' gene_models_checked_test.gff3 | wc -l
```

---

[Home](README.md) | [← EVM consensus & polishing](07-EVM_consensus.md) | [Next: Rename →](09-Renaming.md)
