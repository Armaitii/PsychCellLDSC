# LDSC reference inputs

Local, user-provided S-LDSC reference data. This directory is excluded from Git except for this README. The files are large; do not add them to this repository.

## Where the files were downloaded from

| File / directory in this folder | Downloaded from |
|---|---|
| `1000G_EUR_Phase3_plink/` (from `1000G_Phase3_plinkfiles.tgz`) — `1000G.EUR.QC.{1..22}.{bed,bim,fam}` | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) — *S-LDSC reference files* (Steven Gazal) |
| `weights_hm3_no_MHC/` (from `1000G_Phase3_weights_hm3_no_MHC.tgz`) | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `baseline_v1.2/` (from `1000G_Phase3_baseline_v1.2_ldscores.tgz`) | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `baselineLD_v2.2/` (from `1000G_Phase3_baselineLD_v2.2_ldscores.tgz`) | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `1000G_Phase3_frq/` (from `1000G_Phase3_frq.tgz`) — `1000G.EUR.QC.{chr}.{frq,bed}`-style frequency files | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `w_hm3.snplist` (from `w_hm3.snplist.gz`) — HapMap3 SNP list, 3-column `SNP A1 A2` | [Zenodo 7773502](https://zenodo.org/records/7773502) (mirror of the Alkes Group's `w_hm3.snplist.bz2`) |
| `eur_w_ld_chr/` (from `eur_w_ld_chr.tar.gz`) — non-partitioned LD scores used by `--rg` | [Zenodo 8182036](https://zenodo.org/records/8182036) (mirror of the Alkes Group's `eur_w_ld_chr.tar.bz2`) |
| `hg38ToHg19.over.chain.gz` — UCSC LiftOver chain for the cCREs | [UCSC hgdownload](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/liftOver/hg38ToHg19.over.chain.gz) |

The original Broad location (<https://data.broadinstitute.org/alkesgroup/LDSCORE>) is now requester-pays on Google Cloud (`broad-alkesgroup-public-requester-pays`); the Zenodo copies above are free. The notebook also notes `baselineLF_v2.2.UKB.tar.gz` on the [AWS Open Data Program](https://broad-alkesgroup-ukbb-ld.s3.amazonaws.com/UKBB_LD/baselineLF_v2.2.UKB.tar.gz); it is not used by this workflow.

Unpack commands and the rest of the setup are in Step 2 of the [main README](../../README.md).

## Notes

- **Genome build: hg19/GRCh37.** All of these references are on hg19/GRCh37 — the same build as the GWAS inputs. The hg38 cCRE BEDs are lifted to hg19 by the workflow to match.
- The workflow expects the layout shown above; see `workflow/config.yaml` for the exact prefixes (`plink_prefix`, `hm3_snplist`, `baseline_dir`, `weights_ld_chr`, `frqfile_chr`, `rg_ld_chr`, `chain_file`).
- Use versions compatible with the LDSC checkout and the analysis design. Document sources, versions, citations, and checksums in [`docs/data-sources.md`](../../docs/data-sources.md).
