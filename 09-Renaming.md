[Home](README.md) | [← Filtering](08-Filtering.md)

---

# 09 — Rename

Apply the final standardized gene nomenclature and verify the annotation is complete.

### 9.1. Generate the renamed annotation

```bash
python2 "${SCRIPTS_DIR}/GFF_RenameThemAll.py" \
  "${GENOME_PREFIX}." 1.0 "${GENOME_PREFIX}." \
  gene_models_checked_test.gff3 \
  > "${GENOME_PREFIX}.renamed.gff3" \
  2> "${GENOME_PREFIX}.renaming.err"

awk '$3=="gene"' "${GENOME_PREFIX}.renamed.gff3" | wc -l
```

### 9.2. Final verification

```bash
python2 "${SCRIPTS_DIR}/GFF_extract_features.py" \
  -a "${GENOME_PREFIX}.renamed.gff3" \
  -g "${GENOME}" \
  -p "${GENOME_PREFIX}.renamed_final" \
  -lcmin \
  > "${GENOME_PREFIX}.renamed_final.log" \
  2> "${GENOME_PREFIX}.renamed_final.err"

awk '$3=="gene"' "${GENOME_PREFIX}.renamed_final.gff3" | wc -l

# Convert protein FASTA to tab-delimited for per-protein inspection
awk '/^>/{if(NR>1)printf("\n%s\t", substr($0,2));else printf("%s\t", substr($0,2));next}{printf "%s",$0}END{print ""}' \
  "${GENOME_PREFIX}.renamed_final.protein.fasta" \
  > "${GENOME_PREFIX}.renamed_final.protein.fasta.tab"

# Proteins with stop codon
grep -c "\*" "${GENOME_PREFIX}.renamed_final.protein.fasta.tab"

# Proteins without stop codon
grep -cv "\*" "${GENOME_PREFIX}.renamed_final.protein.fasta.tab"
```

The final annotation file is `${GENOME_PREFIX}.renamed_final.gff3`.

---

[Home](README.md) | [← Filtering](08-Filtering.md)
