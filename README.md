# nanopore-methylation-pipeline
End-to-end ONT nanopore pipeline for DNA modification detection in glioblastoma

# Nanopore DNA Modification Detection Pipeline

End-to-end pipeline for detecting radiation-induced DNA modifications in glioblastoma using Oxford Nanopore Technology (ONT) long-read sequencing.

## Overview

This pipeline takes raw nanopore pod5 files and produces genome-wide DNA modification profiles comparing irradiated samples against unirradiated controls using ELIGOS2.

## Pipeline Steps

1. **Basecalling** — Dorado basecaller with simultaneous methylation calling (5mCG, 5hmCG, 6mA)
2. **Alignment** — minimap2 (via Dorado) to hg38 reference genome
3. **Preprocessing** — `eligos2 map_preprocess` to filter short alignments
4. **Pooling** — `samtools merge` to combine replicates
5. **Modification detection** — `eligos2 pair_diff_mod` comparing irradiated vs mock control

## Scripts

| Script | Description |
|--------|-------------|
| `dorado_basecall.sh` | Basecalling and alignment using Dorado |
| `dorado_mock_rep1.sh` | Basecalling for mock control replicate 1 |
| `dorado_mock_rep2.sh` | Basecalling for mock control replicate 2 |
| `merge_0Gy_mock.sh` | Merge mock control replicates |
| `eligos2_bychrom.sh` | Run ELIGOS2 pair_diff_mod per chromosome |
| `submit_bychrom.sh` | Submit all chromosome jobs for both comparisons |
| `merge_bychrom.sh` | Merge per-chromosome results into genome-wide output |

## Requirements

- Dorado 1.4.0
- samtools
- ELIGOS2 (patched version — see Notes)
- SLURM job scheduler
- hg38 reference genome

## Usage

### 1. Basecalling
```bash
sbatch dorado_basecall.sh
```

### 2. Run ELIGOS2 per chromosome
```bash
# Test on one chromosome first
sbatch eligos2_bychrom.sh chr20 8Gy_3hr

# Submit all chromosomes for both comparisons
bash submit_bychrom.sh
```

### 3. Merge results
```bash
bash merge_bychrom.sh 8Gy_3hr
bash merge_bychrom.sh 4Gy_2wk
```

## Configuration

Before running, set the paths in each script:
```bash
ACCOUNT=your_slurm_account
EMAIL=your_email@university.edu
REFERENCE=/path/to/hg38/hg38.fa
CONDA_ENV=/path/to/eligos2/conda/env
ELIGOS2_SRC=/path/to/eligos2/src
SCRATCH=/path/to/scratch
```

## Notes

- ELIGOS2 requires a patched version with pinned dependencies (python=3.8, rpy2=3.4.5, pandas=1.3.5, numpy=1.21, bedtools=2.25.0) and a modified `_misc.py` that removes the region merging step
- Chromosomes are split into 1MB chunks to keep memory usage ~20GB per job
- Each chromosome job runs independently and can be parallelized

## Samples

| Sample | Dose | Timepoint |
|--------|------|-----------|
| Mock control | 0 Gy | 3 hours |
| Sample 1 | 4 Gy | 3 hours |
| Sample 2 | 8 Gy | 3 hours |
| Sample 3 | 4 Gy | 2 weeks |

## Author

Anantamurthy — Modrek Lab, USC
