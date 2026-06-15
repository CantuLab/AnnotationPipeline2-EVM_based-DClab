# AnnotationPipeline2 — EVM-based (DC Lab)

A step-by-step pipeline for structural genome annotation combining transcript-guided training, ab initio prediction, and evidence-based consensus calling.

## Overview

This pipeline performs structural annotation of a genome assembly using:

- **Repeat masking** — RepeatMasker
- **Transcript alignment** — GMAP and BLAT via PASA
- **Ab initio gene prediction** — Augustus and GeneMark-ET, trained on PASA transcript models
- **Evidence-based consensus** — EVidenceModeler (EVM)
- **Annotation polishing** — PASA update cycle

## Table of contents

| Step | Description |
|---|---|
| [00 — Setup](00-Setup.md) | Prerequisites, tool paths, and variable configuration |
| [01 — External evidences](01-External_evidences.md) | Collecting transcript evidence (Iso-Seq, CDS from related species) |
| [02 — Repeat annotation](02-Repeat_annotation.md) | Genome repeat masking with RepeatMasker |
| [03 — Training set creation](03-Training_set.md) | Transcript alignment and gene model extraction with PASA |
| [04 — Predictor training](04-Predictor_training.md) | Training Augustus and GeneMark-ET on PASA gene models |
| [05 — Ab initio prediction](05-Ab_initio_prediction.md) | Genome-wide gene prediction with Augustus, GeneMark, and PASA |
| [06 — Evidence alignment](06-Evidence_alignment.md) | Preparing transcript alignment tracks for EVM |
| [07 — EVM consensus & polishing](07-EVM_consensus.md) | Evidence-weighted consensus calling and PASA-based annotation polishing |
| [08 — Filtering](08-Filtering.md) | Removing incomplete and short gene models |
| [09 — Renaming](09-Renaming.md) | Applying final standardized gene nomenclature |

## Related resources

This pipeline is a continuation of the original EVM-based annotation pipeline developed in the DC Lab:

> [AnnotationPipeline-EVM_based-DClab](https://github.com/andreaminio/AnnotationPipeline-EVM_based-DClab)

## Citations

This pipeline was used in the following publications:

- Massonnet M, Zaccheo M, Cochetel N, Figueroa-Balderas R, Riaz S, Cantu D (2026). Haplotype graph analysis of *PdR1* uncovers resistance diversity to Pierce's disease in *Vitis arizonica* and its hybrids. *G3 Genes|Genomes|Genetics*, jkag115. https://doi.org/10.1093/g3journal/jkag115

- Massonnet M, Figueroa-Balderas R, Cochetel N, Riaz S, Pap D, Walker MA, Cantu D (2025). Dissection of the *Ren6* and *Ren7* powdery mildew resistance loci in *Vitis piasezkii* DVIT2027 using phased parental–progeny genomes and intraspecific locus graph reconstruction. *G3 Genes|Genomes|Genetics*, 15(12), jkaf250. https://doi.org/10.1093/g3journal/jkaf250
