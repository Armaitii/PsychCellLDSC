# LDSC reference inputs

This directory holds local, user-provided reference data and is excluded from Git except for this README. The current `workflow/config.yaml` expects:

- `hg38ToHg19.over.chain.gz` — UCSC hg38-to-hg19 chain file.
- `1000G_EUR_Phase3_plink/1000G.EUR.QC.{chr}.{bed,bim,fam}` for chromosomes 1–22.
- `w_hm3.snplist` — HapMap3 SNP list.
- `baseline_v1.2/` and `baselineLD_v2.2/` — baseline references.
- `weights_hm3_no_MHC/`, `1000G_Phase3_frq/`, and `eur_w_ld_chr/` — weights, allele frequencies, and non-partitioned LD scores.

Use versions compatible with the LDSC checkout and analysis design. Document sources, versions, citations, and checksums in `docs/data-sources.md`. These panels are large; do not add them to this Git repository unless their terms and hosting plan explicitly permit it.
