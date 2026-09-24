# Nanopore DNA Modification Detection Pipeline

End-to-end pipeline for detecting radiation-induced DNA modifications in glioblastoma using Oxford Nanopore Technology (ONT) long-read sequencing.

## Overview

This pipeline takes raw nanopore pod5 files and produces genome-wide DNA modification profiles comparing irradiated glioblastoma stem cells (GSC20) against unirradiated controls using ELIGOS2.

## Pipeline Steps

1. **Basecalling** — Dorado basecaller with simultaneous methylation calling (5mCG, 5hmCG, 6mA)
2. **Alignment** — minimap2 (via Dorado) to hg38 reference genome
3. **Preprocessing** — `eligos2 map_preprocess` to filter short alignments (< 200bp)
4. **Pooling** — `samtools merge` to combine replicates where applicable
5. **Modification detection** — `eligos2 pair_diff_mod` comparing irradiated vs mock control per chromosome (both + and - strands, 1Mbp chunks)
6. **Filtering** — filter by pval < 0.001, oddR > 1.2, min_depth = 5, homopolymer removed, A/G/T only
7. **Merging** — combine per-chromosome filtered results into genome-wide output

## Scripts

| Script | Description |
|---|---|
| `dorado_basecall.sh` | Basecalling for single samples (one pod5 directory) |
| `dorado_mock.sh` | Basecalling for mock control with two replicates — basecalls rep1 and rep2 separately then merges |
| `eligos2_bychrom.sh` | Main ELIGOS2 worker — runs pair_diff_mod for one chromosome, both strands, 1Mbp chunks |
| `submit_bychrom.sh` | Submits all 25 chromosome jobs for a given comparison |
| `eligos2_filter.sh` | Filters ELIGOS2 results by pval, oddR, homopolymer, base type |
| `merge_filtered.sh` | Merges per-chromosome filtered files into one genome-wide txt file |

## Samples (GSC20)

| Sample | Dose | Timepoint | Replicates | Status |
|---|---|---|---|---|
| Mock control | 0 Gy | 3 hours | 2 (merged) | ✓ Complete |
| 8Gy 3hr | 8 Gray | 3 hours | 1 | ✓ Complete |
| 4Gy 2wk | 4 Gray | 2 weeks | 2 (merged) | ✓ Complete |
| 4Gy 3hr | 4 Gray | 3 hours | 2 (merged) | ⏳ Dorado running |
| 2Gy 3hr | 2 Gray | 3 hours | 1 | ✗ Failed sequencing — needs resequencing |
| 0Gy 2023 | 0 Gray | 3 hours | 1 | ✓ Basecalled |

## Comparisons Run

| Comparison | Candidates (pval<0.001, oddR>1.2) |
|---|---|
| 8Gy 3hr vs 0Gy mock | 2,793,122 |
| 4Gy 2wk vs 0Gy mock | 1,756,593 |
| Shared positions | 173,987 |

## Filtering Parameters

```
pval      < 0.001
oddR      > 1.2
min_depth = 5
homopolymer regions removed
bases: A, G, T only (C excluded — epigenetic methylation called separately by Dorado)
both strands analyzed (+ and -)
```

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

### 2. Preprocessing

```bash
eligos2 map_preprocess -i sample.bam -aln 200 -o sample.preprocess.bam -t 8
```

### 3. Run ELIGOS2 per chromosome

Test on one chromosome first:
```bash
sbatch eligos2_bychrom.sh chr20 8Gy_3hr
```

Submit all 25 chromosomes:
```bash
bash submit_bychrom.sh
```

### 4. Filter results

```bash
sbatch eligos2_filter.sh 8Gy_3hr
sbatch eligos2_filter.sh 4Gy_2wk
```

### 5. Merge into genome-wide file

```bash
bash merge_filtered.sh /path/to/8Gy_3hr_filtered_pval0.001_oddR1.2
bash merge_filtered.sh /path/to/4Gy_2wk_filtered_pval0.001_oddR1.2
```

## Configuration

Each script has a configuration section at the top. Set these variables before running:

```bash
ACCOUNT=your_slurm_account
EMAIL=your_email@university.edu
REFERENCE=/path/to/hg38/hg38.fa
CONDA_ENV=/path/to/eligos2/conda/env
ELIGOS2_SRC=/path/to/eligos2/patched/src
SCRATCH=/path/to/scratch2
LOG_DIR=/path/to/logs
OUT_BASE=/path/to/output
```

## Notes

### ELIGOS2 Patch Requirements

ELIGOS2 requires a patched version with:

- Pinned dependencies: `python=3.8`, `rpy2=3.4.5`, `pandas=1.3.5`, `numpy=1.21`, `bedtools=2.25.0`
- Modified `_misc.py` — remove the `.merge()` call in `readBed()`:

```python
# Change this:
mergedBed = beds.sort().merge(s=True, c='4', o='distinct')
# To this:
mergedBed = beds.sort()
```

### Both Strand Analysis

The pipeline analyzes modifications on both DNA strands (+ and -). The BED file generator in `eligos2_bychrom.sh` creates entries for both strands per chunk:

```
chr1  0  1000000  +  chr1_chunk1_pos
chr1  0  1000000  -  chr1_chunk1_neg
```

### Memory and Chunking

Each chromosome is split into 1Mbp chunks:
- Each 1Mbp chunk uses ~2.8GB per worker
- With 8 threads: ~22GB total — fits on standard HPC nodes (94GB)
- chr1 requires ~128GB due to size (~249Mbp, 498 chunks with both strands)

### GPU Compatibility

Dorado requires a V100 or newer GPU. Request specifically:
```bash
#SBATCH --gres=gpu:v100:1
```

### Dorado Models (v5.2.0)

```
Basecalling:  dna_r10.4.1_e8.2_400bps_hac@v5.2.0
Methylation:  dna_r10.4.1_e8.2_400bps_hac@v5.2.0_5mCG_5hmCG@v2
              dna_r10.4.1_e8.2_400bps_hac@v5.2.0_6mA@v1
```

## Author

Anantamurthy Rajesh — Modrek Lab, USC
