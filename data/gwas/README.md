# GWAS inputs

Local, user-provided GWAS summary statistics. This directory is excluded from Git except for this README. Place raw and already-munged files in the subdirectories referenced by [`workflow/config.yaml`](../../workflow/config.yaml) (`OCD`, `SCZ`, `TS`, and `external/`); paths in that config are relative to the `workflow/` working directory. Do not rename inputs without updating the config.

**Genome build: hg19/GRCh37.** The GWAS summary statistics use hg19/GRCh37 coordinates and rsIDs, matching the 1000 Genomes Phase 3 LDSC references.

## Where the files were downloaded from

| Trait | Example file | Format | Downloaded from |
|---|---|---|---|
| OCD | `OCD/daner_OCD_full_wo23andMe_190522.gz` | daner | Psychiatric Genomics Consortium download page: <https://pgc.unc.edu/for-researchers/download-results/>. Meta-analysis **excluding 23andMe** from [*Genome-wide analyses identify 30 loci associated with obsessive–compulsive disorder*](https://www.nature.com/articles/s41588-025-02189-z). |
| SCZ | `SCZ/daner_PGC_SCZ52_0513a.hq2.gz` | daner | PGC 2014, [*Biological Insights From 108 Schizophrenia-Associated Genetic Loci*](https://doi.org/10.1038/nature13595); [figshare scz2014](https://figshare.com/articles/dataset/scz2014/14672163). |
| TS | `TS/TS_Oct2018.gz` | simple | Yu et al. 2019, [*Interrogating the Genetic Determinants of Tourette's Syndrome and Other Tic Disorders Through GWAS*](https://doi.org/10.1176/appi.ajp.2018.18070857); [figshare ts2019](https://figshare.com/articles/dataset/ts2019/14672232). |
| ADHD | `external/PASS.Attention_Deficit_Hyperactivity_Disorder_ADHD.Demontis2023.sumstats.gz` | premunged | Already-munged sumstats (Demontis 2023). |
| Anorexia | `external/PASS.Anorexia_Nervosa.Watson2019.sumstats.gz` | premunged | Already-munged sumstats (Watson 2019). |
| rg comparison traits | `external/PASS.<Trait>.<Study>.sumstats.gz` | premunged | Already-munged external collections referenced in `workflow/config.yaml` (Price-lab bundle / traits_indep107). |

The full provenance inventory — including the other configured `rg_comparison_traits` (Depression, Bipolar, Schizophrenia 2022, Insomnia, BMI, Height, Alcohol, Cigarettes, Endometriosis, Lupus, Physical_Activity, Reaction_Time, Sleep_Duration) — is in [`docs/data-sources.md`](../../docs/data-sources.md).

## Input formats (declared per trait in the config)

- `daner` — standard PGC daner file with per-SNP `FRQ_A`/`FRQ_U` and N columns. The workflow's `prep_gwas` rule computes per-SNP `Neff` and `MAF` automatically; no manual preparation is needed.
- `simple` — minimal fixed-N format (e.g. PGC-TS Yu et al. 2019: `SNP CHR BP A1 A2 INFO OR SE P`); requires `n_cas`/`n_con` in the config.
- `premunged` — already-munged, LDSC-compatible sumstats used directly by the h² regression; no `raw_file` needed.

Derived munged and rg-filtered sumstats are written under `workflow/gwas/` and are ignored by Git, as is the optional `exclude_snps.txt` list.

## Terms and provenance

Verify the access and redistribution terms for every study; do not commit controlled or otherwise restricted data. Record source, version, citation, and checksum information in [`docs/data-sources.md`](../../docs/data-sources.md) without copying restricted data into this repository.
