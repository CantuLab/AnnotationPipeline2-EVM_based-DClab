[Home](README.md) | [← Evidence alignment](06-Evidence_alignment.md) | [Next: Filtering →](08-Filtering.md)

---

# 07 — EVM consensus & annotation polishing

This step has two phases:

1. **EVM consensus** — integrates ab initio predictions, transcript alignments, and repeat coordinates into a weighted consensus annotation
2. **PASA polishing** — compares the EVM gene models against assembled transcripts and updates gene structures (corrects splice sites, adds UTRs, generates isoforms where supported by transcript evidence)

---

## 7.1. EVM setup

```bash
cd "${EVM_DIR}"
cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")

# Merge all ab initio predictions and validate file format
cat prediction.*.gff3 > predictions.gff3
"${EVM_TOOLS}/EvmUtils/gff3_gene_prediction_file_validator.pl" predictions.gff3

# Merge all transcript alignment tracks
cat transcript_alignment.*.gff3 > transcript_alignments.gff3
```

Create the evidence weights file. Higher weights give more influence to that evidence type. Adjust based on your confidence in each source:

```bash
echo -e "ABINITIO_PREDICTION\tAUGUSTUS\t9\n\
ABINITIO_PREDICTION\tGeneMark.hmm3\t9\n\
TRANSCRIPT\tgmap\t7\n\
TRANSCRIPT\tBLAT\t6\n\
TRANSCRIPT\tPASA_assemblies\t20\n\
OTHER_PREDICTION\tPASA_transdecoder\t25" > weights.txt
```

## 7.2. Run EVM

```bash
# 1. Partition the genome into overlapping segments for parallel processing
"${EVM_TOOLS}/EvmUtils/partition_EVM_inputs.pl" \
  --partition_dir ./partition_dir \
  --genome "${GENOME}" \
  --gene_predictions predictions.gff3 \
  --transcript_alignments transcript_alignments.gff3 \
  --repeats repeats.gff3 \
  --segmentSize 300000 \
  --overlapSize 100000 \
  --partition_listing partitions_list.out \
  &> 1-partition.log

# 2. Generate and run EVM commands in parallel
"${EVM_TOOLS}/EvmUtils/write_EVM_commands.pl" \
  --genome "${GENOME}" \
  --weights "$(pwd)/weights.txt" \
  --gene_predictions predictions.gff3 \
  --transcript_alignments transcript_alignments.gff3 \
  --repeats repeats.gff3 \
  --output_file_name evm.out \
  --partitions "$(pwd)/partitions_list.out" \
  > commands.list \
  && parallel -j ${THREADS} :::: commands.list

# 3. Merge partition results
"${EVM_TOOLS}/EvmUtils/recombine_EVM_partial_outputs.pl" \
  --partitions "$(pwd)/partitions_list.out" \
  --output_file_name evm.out

# 4. Convert to GFF3
"${EVM_TOOLS}/EvmUtils/convert_EVM_outputs_to_GFF3.pl" \
  --partitions "$(pwd)/partitions_list.out" \
  --output evm.out \
  --genome "${GENOME}"

# 5. Concatenate all partition GFF3 files
find "$(pwd)" -regex ".*evm.out.gff3" -exec cat {} \; > EVM.all.gff3
```

Filter low-quality gene models (incomplete CDS, models with internal stops, etc.):

```bash
python2 "${SCRIPTS_DIR}/GFF_extract_features.py" \
  -l -c -i -n \
  -p EVM.filtered \
  -a EVM.all.gff3 \
  -g "${GENOME}" \
  > EVM.filtering.log
```

---

## 7.3. Annotation polishing with PASA

```bash
cd "${POLISHING_DIR}"
cp "${EVM_DIR}/EVM.filtered.gff3" .
cp "${TRAINING_DIR}/all_transcripts.fasta"* .
cp "${GENOME_FASTA}" .
GENOME=$(basename "${GENOME_FASTA}")
PASA_DB="${POLISHING_DIR}/polishing_PASA.sqlite"
```

Create `pasa_config.txt` — replace `<POLISHING_DIR>` with your actual path:

```
DATABASE=<POLISHING_DIR>/polishing_PASA.sqlite

run_spliced_aligners.pl:-N=5

validate_alignments_in_db.dbi:--MIN_PERCENT_ALIGNED=90
validate_alignments_in_db.dbi:--MIN_AVG_PER_ID=85
validate_alignments_in_db.dbi:--MAX_INTRON_LENGTH=45000

subcluster_builder.dbi:-m=30

cDNA_annotation_comparer.dbi:--MIN_PERCENT_OVERLAP=50
cDNA_annotation_comparer.dbi:--MIN_PERCENT_PROT_CODING=30
cDNA_annotation_comparer.dbi:--MIN_PERID_PROT_COMPARE=50
cDNA_annotation_comparer.dbi:--MIN_PERCENT_LENGTH_FL_COMPARE=70
cDNA_annotation_comparer.dbi:--MIN_PERCENT_LENGTH_NONFL_COMPARE=70
cDNA_annotation_comparer.dbi:--MIN_PERCENT_ALIGN_LENGTH=70
cDNA_annotation_comparer.dbi:--MIN_PERCENT_OVERLAP_GENE_REPLACE=90
cDNA_annotation_comparer.dbi:--MAX_UTR_EXONS=5
```

Set environment and run the initial alignment phase:

```bash
export GMAPHOME="${GMAP_DIR}"
export PATH="${GMAP_DIR}/bin:${GMAP_DIR}:${PATH}"
export BLATHOME="${BLAT_DIR}"
export PATH="${BLAT_DIR}:${PATH}"
export PASAHOME="${PASA_DIR}"
export PATH="${PASA_DIR}/bin:${PASA_DIR}:${PATH}"

mkdir -p pasa_run.log.dir

"${PASA_DIR}/scripts/create_sqlite_cdnaassembly_db.dbi" \
  -c pasa_config.txt \
  -S "${PASA_DIR}/schema/cdna_alignment_sqliteschema" -r \
  2> pasa_run.log.dir/1_create_db.err

samtools faidx "${GENOME}" 2> pasa_run.log.dir/1b_faidx.err

"${GMAP_DIR}/bin/gmap_build" \
  -D "${POLISHING_DIR}" -d "${GENOME}.gmap" -k 13 \
  "${POLISHING_DIR}/${GENOME}" >&2

"${PASA_DIR}/scripts/run_spliced_aligners.pl" \
  --aligners gmap,blat \
  --genome "${POLISHING_DIR}/${GENOME}" \
  --transcripts "${POLISHING_DIR}/all_transcripts.fasta" \
  -I 45000 -N 5 --CPU ${THREADS} \
  2> pasa_run.log.dir/2_run_spliced_aligners.err
```

Save as `PASA_polishing.sh` and run:

```bash
#!/bin/bash
PASA_DB="${POLISHING_DIR}/polishing_PASA.sqlite"
GENOME_PATH="${POLISHING_DIR}/$(basename ${GENOME_FASTA})"

"${PASA_DIR}/scripts/upload_transcript_data.dbi" \
  -M "${PASA_DB}" -t "${POLISHING_DIR}/all_transcripts.fasta" -f NULL \
  2> pasa_run.log.dir/3_upload_transcript_data.err

"${PASA_DIR}/scripts/import_spliced_alignments.dbi" -M "${PASA_DB}" -A gmap -g gmap.spliced_alignments.gff3 \
  2> pasa_run.log.dir/4_import_gmap.err
"${PASA_DIR}/scripts/import_spliced_alignments.dbi" -M "${PASA_DB}" -A blat -g blat.spliced_alignments.gff3 \
  2> pasa_run.log.dir/5_import_blat.err

"${PASA_DIR}/scripts/update_fli_status.dbi" -M "${PASA_DB}" \
  -f all_transcripts.fasta.transdecoder.gff3.fl_accs \
  2> pasa_run.log.dir/6_update_fli_status.err

"${PASA_DIR}/scripts/validate_alignments_in_db.dbi" \
  -M "${PASA_DB}" -g "${GENOME_PATH}" -t "${POLISHING_DIR}/all_transcripts.fasta" \
  --MAX_INTRON_LENGTH 45000 --CPU ${THREADS} \
  --MIN_PERCENT_ALIGNED 90 --MIN_AVG_PER_ID 85 \
  > alignment.validations.output \
  2> pasa_run.log.dir/7_validate_alignments.err

"${PASA_DIR}/scripts/update_alignment_status.dbi" -M "${PASA_DB}" \
  < alignment.validations.output \
  > pasa_run.log.dir/alignment.validation_loading.output \
  2> pasa_run.log.dir/8_update_alignment_status.err

"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -v -A -P gmap \
  > polishing_PASA.sqlite.valid_gmap_alignments.gff3  2> pasa_run.log.dir/9_gff3_valid_gmap.err
"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -f -A -P gmap \
  > polishing_PASA.sqlite.failed_gmap_alignments.gff3 2> pasa_run.log.dir/10_gff3_failed_gmap.err
"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -v -A -P blat \
  > polishing_PASA.sqlite.valid_blat_alignments.gff3  2> pasa_run.log.dir/11_gff3_valid_blat.err
"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -f -A -P blat \
  > polishing_PASA.sqlite.failed_blat_alignments.gff3 2> pasa_run.log.dir/12_gff3_failed_blat.err

"${PASA_DIR}/scripts/assign_clusters_by_stringent_alignment_overlap.dbi" -M "${PASA_DB}" -L 30 \
  > pasa_run.log.dir/cluster_reassignment_by_stringent_overlap.out \
  2> pasa_run.log.dir/13_assign_clusters.err

"${PASA_DIR}/scripts/assemble_clusters.dbi" \
  -G "${GENOME_PATH}" -M "${PASA_DB}" -T ${THREADS} \
  > polishing_PASA.sqlite.pasa_alignment_assembly_building.ascii_illustrations.out \
  2> pasa_run.log.dir/14_assemble_clusters.err

"${PASA_DIR}/scripts/assembly_db_loader.dbi" -M "${PASA_DB}" \
  > pasa_run.log.dir/alignment_assembly_loading.out \
  2> pasa_run.log.dir/15_assembly_db_loader.err

"${PASA_DIR}/scripts/subcluster_builder.dbi" -G "${GENOME_PATH}" -M "${PASA_DB}" -m 30 \
  > pasa_run.log.dir/alignment_assembly_subclustering.out \
  2> pasa_run.log.dir/16_subcluster_builder.err

"${PASA_DIR}/scripts/populate_mysql_assembly_alignment_field.dbi" -M "${PASA_DB}" -G "${GENOME_PATH}" \
  2> pasa_run.log.dir/17_populate_alignment_field.err
"${PASA_DIR}/scripts/populate_mysql_assembly_sequence_field.dbi"  -M "${PASA_DB}" -G "${GENOME_PATH}" \
  2> pasa_run.log.dir/17b_populate_sequence_field.err

"${PASA_DIR}/scripts/subcluster_loader.dbi" -M "${PASA_DB}" \
  < pasa_run.log.dir/alignment_assembly_subclustering.out \
  2> pasa_run.log.dir/18_subcluster_loader.err

"${PASA_DIR}/scripts/alignment_assembly_to_gene_models.dbi" -M "${PASA_DB}" -G "${GENOME_PATH}" \
  2> pasa_run.log.dir/19_alignment_assembly_to_gene_models.err

"${PASA_DIR}/scripts/PASA_transcripts_and_assemblies_to_GFF3.dbi" -M "${PASA_DB}" -a \
  > polishing_PASA.sqlite.pasa_assemblies.gff3 \
  2> pasa_run.log.dir/20_pasa_assemblies_to_gff3.err

"${PASA_DIR}/scripts/describe_alignment_assemblies_cgi_convert.dbi" -M "${PASA_DB}" \
  > polishing_PASA.sqlite.pasa_assemblies_described.txt \
  2> pasa_run.log.dir/21_describe_assemblies.err

"${PASA_DIR}/scripts/Load_Current_Gene_Annotations.dbi" \
  -c pasa_config.txt -g "${GENOME_PATH}" -P EVM.filtered.gff3 \
  > pasa_run.log.dir/output.annot_loading.out \
  2> pasa_run.log.dir/22_load_current_gene_annotations.err

export PATH="${FASTA_BIN}:${PATH}"

"${PASA_DIR}/scripts/cDNA_annotation_comparer.dbi" \
  -G "${GENOME_PATH}" --CPU ${THREADS} -M "${PASA_DB}" \
  --MIN_PERCENT_PROT_CODING 30 \
  --MIN_PERCENT_LENGTH_NONFL_COMPARE 70 \
  --MAX_UTR_EXONS 5 \
  --MIN_PERCENT_LENGTH_FL_COMPARE 70 \
  --MIN_PERCENT_ALIGN_LENGTH 70 \
  --MIN_PERCENT_OVERLAP_GENE_REPLACE 90 \
  --MIN_PERID_PROT_COMPARE 50 \
  --MIN_PERCENT_OVERLAP 50 \
  > pasa_run.log.dir/polishing_PASA.sqlite.annotation_compare.out \
  2> pasa_run.log.dir/23_cDNA_annotation_comparer.err

"${PASA_DIR}/scripts/dump_valid_annot_updates.dbi" \
  -M "${PASA_DB}" -V -R -g "${GENOME_PATH}" \
  > polishing_PASA.sqlite.gene_structures_post_PASA_updates.gff3 \
  2> pasa_run.log.dir/24_dump_valid_annot_updates.err

python2 "${SCRIPTS_DIR}/GFF_RenameThemAll.py" \
  "${GENOME_PREFIX}." 1.0 "${GENOME_PREFIX}." \
  polishing_PASA.sqlite.gene_structures_post_PASA_updates.gff3 \
  > polishing_PASA.sqlite.gene_structures_post_PASA_updates.renamed.gff3 \
  2> polishing_PASA.sqlite.gene_structures_post_PASA_updates.renaming.err

python2 "${SCRIPTS_DIR}/GFF_extract_features.py" \
  -p EVM.filtered.polished.clean \
  -a "${POLISHING_DIR}/polishing_PASA.sqlite.gene_structures_post_PASA_updates.renamed.gff3" \
  -g "${POLISHING_DIR}/${GENOME_PATH}" \
  > EVM.filtered.polished.cleaning.log \
  2> pasa_run.log.dir/25_GFF_extract_features.err

# Move to filtering directory with cleaner names
mkdir -p "${FILTERING_DIR}"
for f in EVM.filtered.polished.clean*.fasta EVM.filtered.polished.clean*.gff3; do
  cp "$f" "${FILTERING_DIR}/${f/EVM.filtered.polished.clean/gene_models}"
done
```

Count genes after polishing:

```bash
awk '$3=="gene"' EVM.filtered.polished.clean.gff3 | wc -l
```

---

[Home](README.md) | [← Evidence alignment](06-Evidence_alignment.md) | [Next: Filtering →](08-Filtering.md)
