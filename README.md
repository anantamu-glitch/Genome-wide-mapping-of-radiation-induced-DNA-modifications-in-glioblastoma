# Nanopore DNA Modification Detection Pipeline

End-to-end pipeline for detecting radiation-induced DNA modifications in glioblastoma using Oxford Nanopore Technology (ONT) long-read sequencing.

## Overview

This pipeline takes raw nanopore pod5 files and produces genome-wide DNA modification profiles comparing irradiated samples against unirradiated controls using ELIGOS2.

## Pipeline Steps

1. **Basecalling** — Dorado basecaller with simultaneous methylation calling (5mCG, 5hmCG, 6mA)
2. **Alignment** — minimap2 (via Dorado) to hg38 reference genome
3. **Preprocessing** — `eligos2 map_preprocess` to filter short alignments (< 200bp)
4. **Pooling** — `samtools merge` to combine replicates where applicable
5. **Modification detection** — `eligos2 pair_diff_mod` comparing irradiated vs mock control per chromosome
6. **Merging** — combine per-chromosome results into genome-wide output

## Scripts

| Script | Description |
|--------|-------------|
| `dorado_basecall.sh` | Basecalling for single samples (one pod5 directory) |
| `dorado_mock.sh` | Basecalling for mock control with two replicates — basecalls rep1 and rep2 separately then merges into one BAM |
| `eligos2_bychrom.sh` | Main ELIGOS2 worker — runs pair_diff_mod for one chromosome split into 1MB chunks |
| `submit_bychrom.sh` | Submits all chromosome jobs for both comparisons (50 jobs total) |
| `merge_bychrom.sh` | Merges per-chromosome results into one genome-wide output file |

## Why Two Dorado Scripts?

- **`dorado_basecall.sh`** is used for samples with a single sequencing run (e.g. 8Gy 3hr, 4Gy 3hr, 4Gy 2wk rep1, 4Gy 2wk rep2)
- **`dorado_mock.sh`** is used specifically for the mock control which has two replicates (rep1 and rep2). It basecalls both replicates sequentially and merges them into one final BAM to maximize coverage for the control

## Requirements

- Dorado 1.4.0
- samtools
- ELIGOS2 (patched version — see Notes below)
- Python 3.8+
- SLURM job scheduler
- hg38 reference genome with `.fai` index

## Usage

### 1. Basecalling

For each irradiated sample:
```bash
sbatch dorado_basecall.sh
```

For the mock control (two replicates):
```bash
sbatch dorado_mock.sh
```

### 2. Preprocessing (run after basecalling)
```bash
eligos2 map_preprocess -i sample.bam -aln 200 -o sample.preprocess.bam -t 8
```

### 3. Run ELIGOS2 per chromosome

Test on one chromosome first:
```bash
sbatch eligos2_bychrom.sh chr20 8Gy_3hr
```

Submit all chromosomes for both comparisons (50 jobs total):
```bash
bash submit_bychrom.sh
```

### 4. Merge results
```bash
bash merge_bychrom.sh 8Gy_3hr
bash merge_bychrom.sh 4Gy_2wk
```

## Configuration

Each script has a configuration section at the top. Set these variables before running:

```bash
ACCOUNT=your_slurm_account
EMAIL=your_email@university.edu
REFERENCE=/path/to/hg38/hg38.fa
CONDA_ENV=/path/to/eligos2/conda/env
ELIGOS2_SRC=/path/to/eligos2/patched/src
SCRATCH=/path/to/scratch
LOG_DIR=/path/to/logs
OUT_BASE=/path/to/output
```

## Samples

| Sample | Dose | Timepoint | Replicates |
|--------|------|-----------|------------|
| Mock control | 0 Gy | 3 hours | 2 (merged) |
| Sample 1 | 4 Gy | 3 hours | 1 |
| Sample 2 | 8 Gy | 3 hours | 1 |
| Sample 3 | 4 Gy | 2 weeks | 2 (merged) |

## Notes

### ELIGOS2 Patch Requirements
ELIGOS2 requires a patched version with:
- Pinned dependencies: `python=3.8`, `rpy2=3.4.5`, `pandas=1.3.5`, `numpy=1.21`, `bedtools=2.25.0`
- Modified `_misc.py` — remove the `.merge()` call in `readBed()` to prevent adjacent BED regions from being collapsed into one:

```python
# Change this:
mergedBed = beds.sort().merge(s=True, c='4', o='distinct')
# To this:
mergedBed = beds.sort()
```

### Memory and Chunking
ELIGOS2's error extraction loads each BED region into memory as one worker process. Without chunking, a whole chromosome uses ~96GB and causes OOM errors. This pipeline splits each chromosome into 1MB chunks:
- Each 1MB chunk uses ~2.8GB per worker (confirmed via testing)
- With 8 threads: ~22GB total — fits on standard HPC nodes
- chr20 (64MB) → 65 chunks, ~7 hours runtime

### GPU Compatibility
Dorado requires a V100 or newer GPU. Older GPUs (e.g. P100) are not compatible with Dorado's CUDA kernels. Request a V100 specifically in your SLURM script:
```bash
#SBATCH --gres=gpu:v100:1
```

## Author

Anantamurthy Rajesh — Modrek Lab, USC
