[Home](README.md) | [← Setup](00-Setup.md) | [Next: Repeat annotation →](02-Repeat_annotation.md)

---

# 01 — External evidences

Collect all external transcript evidences into one directory before starting the pipeline. Supported types include Iso-Seq HQ reads and CDS FASTA files from closely related species. All files must be in FASTA format.

```bash
# Copy or symlink your evidence files into the external evidence directory
# cp /path/to/isoseq_HQ.fasta "${EXTERNAL_DIR}/"
# cp /path/to/related_species.CDS.fasta "${EXTERNAL_DIR}/"
```

These files will be merged and used in [step 03 (Training set creation)](03-Training_set.md) and re-used in [step 07 (EVM consensus & polishing)](07-EVM_consensus.md).

---

[Home](README.md) | [← Setup](00-Setup.md) | [Next: Repeat annotation →](02-Repeat_annotation.md)
