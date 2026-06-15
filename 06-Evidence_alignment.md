[Home](README.md) | [← Ab initio prediction](05-Ab_initio_prediction.md) | [Next: EVM consensus & polishing →](07-EVM_consensus.md)

---

# 06 — Evidence alignment

The GMAP and BLAT transcript alignments computed during [step 03 (Training set creation)](03-Training_set.md) are reused here as transcript alignment evidence for EVM. No new alignments are run in this step.

```bash
cd "${TRAINING_DIR}"
GENOME=$(basename "${GENOME_FASTA}")

# GMAP alignments — fix the database path in the source column
sed "s:${GENOME}.gmap:gmap:" gmap.spliced_alignments.gff3 \
  > "${EVM_DIR}/transcript_alignment.gmap.gff3"

# BLAT alignments
cp blat.spliced_alignments.gff3 "${EVM_DIR}/transcript_alignment.blat.gff3"

# PASA assembled transcripts
cat mRNA_training_PASA.sqlite.pasa_assemblies.gff3 \
  | sed 's:assembler-mRNA_training_PASA.sqlite:PASA_assemblies:' \
  > "${EVM_DIR}/transcript_alignment.pasa.gff3"
```

---

[Home](README.md) | [← Ab initio prediction](05-Ab_initio_prediction.md) | [Next: EVM consensus & polishing →](07-EVM_consensus.md)
