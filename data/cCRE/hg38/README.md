# CATlas cCRE input (hg38)

Place one BED file per unique cell type listed in `workflow/config.yaml` here, using the filename pattern:

```text
<CELLTYPE>.bed.txt
```

For example, `MSN` is expected at `MSN.bed.txt`. The workflow reads this directory through `ccre_hg38_dir: ../data/cCRE/hg38` when Snakemake is launched from `workflow/`. Source cCRE files are excluded from Git; verify CATlas terms and cite the source/version used.
