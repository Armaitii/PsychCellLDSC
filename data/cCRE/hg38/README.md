# Cell-type cCRE inputs (hg38)

Local, user-provided cis-regulatory element (cCRE) annotations, one BED file per configured cell type. This directory is excluded from Git except for this README.

## Where the files were downloaded from

The files used in this analysis are human-brain cCREs called from snATAC-seq at cell-type (subclass) level, downloaded from **CATlas**:

- Download: <https://decoder-genetics.wustl.edu/catlasv1/humanbrain/data/cCREs/subclass/>
- Study: [*A comparative atlas of single-cell chromatin accessibility in the human brain*](https://pmc.ncbi.nlm.nih.gov/articles/PMC10852054/)
- Related analysis code: [snATACutils](https://github.com/yal054/snATACutils), [SnapATAC](https://github.com/r3fang/SnapATAC)

## Expected files

Use the filename pattern:

```text
<CELLTYPE>.bed.txt
```

For example, `MSN` is expected at `MSN.bed.txt`. The cell-type list is in [`workflow/config.yaml`](../../../workflow/config.yaml); each `celltypes:` entry needs a matching file here. The workflow reads this directory through `ccre_hg38_dir: ../data/cCRE/hg38` when Snakemake is launched from `workflow/`.

**Genome build: hg38.** The CATlas files are in hg38 coordinates. The workflow lifts them to **hg19** (UCSC `liftOver`, output under [`workflow/cCRE/hg19/`](../../../workflow/cCRE/hg19/)) to match the hg19/GRCh37 LDSC references and GWAS inputs.

Verify CATlas terms and cite the source/version used. Do not commit the source BED files.
