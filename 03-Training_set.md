[Home](README.md) | [← Repeat annotation](02-Repeat_annotation.md) | [Next: Predictor training →](04-Predictor_training.md)

---

# 03 — Training set creation

PASA aligns all transcript evidence to the genome using GMAP and BLAT, assembles overlapping alignments into consensus transcript models, and uses TransDecoder to identify ORFs. The resulting transcript-based gene models are used to train Augustus and GeneMark-ET in [step 04](04-Predictor_training.md). The alignment files produced here are also reused in [steps 05](05-Ab_initio_prediction.md) and [06](06-Evidence_alignment.md).

### 3.0. Setup

```bash
cd "${TRAINING_DIR}"
cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")

# Merge all transcript evidence into one file
cat "${EXTERNAL_DIR}/"*.fasta > all_transcripts.fasta
grep '^>' all_transcripts.fasta | sed 's:>::;s: .*::' > all_transcripts_names.txt
```

Set the environment for PASA and its dependencies:

```bash
export GMAPHOME="${GMAP_DIR}"
export PATH="${GMAP_DIR}/bin:${GMAP_DIR}:${PATH}"
export BLATHOME="${BLAT_DIR}"
export PATH="${BLAT_DIR}:${PATH}"
export PASAHOME="${PASA_DIR}"
export PATH="${PASA_DIR}/bin:${PASA_DIR}:${PATH}"
```

### 3.1. Write the PASA configuration file

Save as `pasa_config.txt`. Replace `<TRAINING_DIR>` and `<MAX_INTRON>` with your actual values (PASA reads this as plain text, not bash):

```
DATABASE=<TRAINING_DIR>/mRNA_training_PASA.sqlite

run_spliced_aligners.pl:-N=5
validate_alignments_in_db.dbi:--MAX_INTRON_LENGTH=<MAX_INTRON>
```

### 3.2. Generate the PASA command list (optional dry-run check)

```bash
"${PASA_DIR}/Launch_PASA_pipeline.pl" \
  -c pasa_config.txt \
  --TRANSDECODER \
  --CPU ${THREADS} \
  --stringent_alignment_overlap 10 \
  -C -r \
  --transcribed_is_aligned_orient \
  --ALIGNERS gmap,blat \
  --transcripts all_transcripts.fasta \
  -f all_transcripts_names.txt \
  -g "${GENOME}" \
  -I ${MAX_INTRON} \
  -x -R \
  > pasa.commands.sh
```

### 3.3. Run PASA — initial alignment steps

```bash
"${PASA_DIR}/scripts/create_sqlite_cdnaassembly_db.dbi" \
  -c pasa_config.txt \
  -S "${PASA_DIR}/schema/cdna_alignment_sqliteschema" \
  -r

samtools faidx "${GENOME}"
samtools faidx all_transcripts.fasta

"${PASA_DIR}/scripts/upload_transcript_data.dbi" \
  -M "${TRAINING_DIR}/mRNA_training_PASA.sqlite" \
  -t "${TRAINING_DIR}/all_transcripts.fasta" \
  -f "${TRAINING_DIR}/all_transcripts_names.txt"

"${GMAP_DIR}/bin/gmap_build" -D ./ -d "${GENOME}.gmap" "${GENOME}"

export GMAPHOME="${GMAP_DIR}"
export PATH="${GMAP_DIR}/bin:${GMAP_DIR}:${PATH}"

"${PASA_DIR}/scripts/run_spliced_aligners.pl" \
  --aligners gmap,blat \
  --genome "${GENOME}" \
  --transcripts "${TRAINING_DIR}/all_transcripts.fasta" \
  -I ${MAX_INTRON} -N 5 --CPU ${THREADS} \
  > mapping.log
```

### 3.4. Run PASA — assembly and TransDecoder

Save as `run.PASA.sh` and execute:

```bash
#!/bin/bash
PASA_DB="${TRAINING_DIR}/mRNA_training_PASA.sqlite"
GENOME_PATH="${TRAINING_DIR}/$(basename ${GENOME_FASTA})"

"${PASA_DIR}/scripts/import_spliced_alignments.dbi" -M "${PASA_DB}" -A gmap -g gmap.spliced_alignments.gff3
"${PASA_DIR}/scripts/import_spliced_alignments.dbi" -M "${PASA_DB}" -A blat -g blat.spliced_alignments.gff3

"${PASA_DIR}/pasa-plugins/transdecoder/TransDecoder.LongOrfs" -t all_transcripts.fasta -S
"${PASA_DIR}/pasa-plugins/transdecoder/TransDecoder.Predict" -t all_transcripts.fasta

"${PASA_DIR}/scripts/extract_FL_transdecoder_entries.pl" all_transcripts.fasta.transdecoder.gff3 \
  > all_transcripts.fasta.transdecoder.gff3.fl_accs

"${PASA_DIR}/scripts/update_fli_status.dbi" \
  -M "${PASA_DB}" -f all_transcripts.fasta.transdecoder.gff3.fl_accs

"${PASA_DIR}/scripts/validate_alignments_in_db.dbi" \
  -M "${PASA_DB}" \
  -g "${GENOME_PATH}" \
  -t all_transcripts.fasta \
  --MAX_INTRON_LENGTH ${MAX_INTRON} --CPU ${THREADS} \
  > alignment.validations.output

"${PASA_DIR}/scripts/update_alignment_status.dbi" -M "${PASA_DB}" \
  < alignment.validations.output \
  > pasa_run.log.dir/alignment.validation_loading.output

"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -v -A -P gmap \
  > mRNA_training_PASA.sqlite.valid_gmap_alignments.gff3
"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -f -A -P gmap \
  > mRNA_training_PASA.sqlite.failed_gmap_alignments.gff3
"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -v -A -P blat \
  > mRNA_training_PASA.sqlite.valid_blat_alignments.gff3
"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -f -A -P blat \
  > mRNA_training_PASA.sqlite.failed_blat_alignments.gff3

"${PASA_DIR}/scripts/set_spliced_orient_transcribed_orient.dbi" -M "${PASA_DB}" \
  > pasa_run.log.dir/setting_aligned_as_transcribed_orientation.output

"${PASA_DIR}/scripts/assign_clusters_by_stringent_alignment_overlap.dbi" -M "${PASA_DB}" -L 10 \
  > pasa_run.log.dir/cluster_reassignment_by_stringent_overlap.out

"${PASA_DIR}/scripts/assemble_clusters.dbi" \
  -G "${GENOME_PATH}" -M "${PASA_DB}" -T 12 \
  > mRNA_training_PASA.sqlite.pasa_alignment_assembly_building.ascii_illustrations.out

"${PASA_DIR}/scripts/assembly_db_loader.dbi" -M "${PASA_DB}" \
  > pasa_run.log.dir/alignment_assembly_loading.out

"${PASA_DIR}/scripts/subcluster_builder.dbi" -G "${GENOME_PATH}" -M "${PASA_DB}" \
  > pasa_run.log.dir/alignment_assembly_subclustering.out

"${PASA_DIR}/scripts/populate_mysql_assembly_alignment_field.dbi" -M "${PASA_DB}" -G "${GENOME_PATH}"
"${PASA_DIR}/scripts/populate_mysql_assembly_sequence_field.dbi"  -M "${PASA_DB}" -G "${GENOME_PATH}"

"${PASA_DIR}/scripts/subcluster_loader.dbi" -M "${PASA_DB}" \
  < pasa_run.log.dir/alignment_assembly_subclustering.out

"${PASA_DIR}/scripts/alignment_assembly_to_gene_models.dbi" -M "${PASA_DB}" -G "${GENOME_PATH}"

"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -a \
  > mRNA_training_PASA.sqlite.pasa_assemblies.gff3

"${PASA_DIR}/scripts/describe_alignment_assemblies_cgi_convert.dbi" -M "${PASA_DB}" \
  > mRNA_training_PASA.sqlite.pasa_assemblies_described.txt
```

### 3.5. Extract training gene models from the genome

```bash
"${PASA_DIR}/scripts/pasa_asmbls_to_training_set.dbi" \
  --pasa_transcripts_fasta mRNA_training_PASA.sqlite.assemblies.fasta \
  --pasa_transcripts_gff3 mRNA_training_PASA.sqlite.pasa_assemblies.gff3
```

### 3.6. Clean the training set GFF3

Removes redundant UTRs, multi-copy entries, and incomplete models:

```bash
python2 "${SCRIPTS_DIR}/GFF_extract_features.py" \
  -g "${GENOME}" \
  -a mRNA_training_PASA.sqlite.assemblies.fasta.transdecoder.genome.gff3 \
  -p mRNA_training_PASA.sqlite.assemblies.fasta.transdecoder.genome.clean \
  -nlcmi \
  > GFF_extract_features.log 2> GFF_extract_features.err
```

---

[Home](README.md) | [← Repeat annotation](02-Repeat_annotation.md) | [Next: Predictor training →](04-Predictor_training.md)
