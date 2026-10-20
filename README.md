# PsychCellLDSC

> *Still under development.*

A Snakemake workflow for **cell-type partitioned heritability** and **genetic-correlation** analyses with stratified LD Score Regression (S-LDSC). It takes cell-type cis-regulatory elements (cCREs) called from snATAC-seq data and GWAS summary statistics, lifts the cCRE annotations from **hg38** to **hg19** to match the LDSC references, computes LD scores once per cell type, runs partitioned h² and `--rg` analyses for one or more traits, and gives out enriched cell types and correlation analyses for each trait. The pipeline can be run automatically on new cell types and traits after updating the config.

## Table of contents

- [Repository layout](#repository-layout)
- [Requirements](#requirements)
- [Getting started](#getting-started)
  - [Step 1 — Clone and set up LDSC (Python 3.9 branch)](#step-1--clone-and-set-up-ldsc-python-39-branch)
  - [Step 2 — Download the LDSC reference files](#step-2--download-the-ldsc-reference-files)
  - [Step 3 — Provide the input GWAS and cCRE files](#step-3--provide-the-input-gwas-and-ccre-files)
  - [Step 4 — Configure the cell types and trait(s) of interest](#step-4--configure-the-cell-types-and-traits-of-interest)
  - [Step 5 — Run the Snakemake workflow](#step-5--run-the-snakemake-workflow)
- [Workflow stages](#workflow-stages)
- [Outputs](#outputs)
- [Data sources and provenance](#data-sources-and-provenance)
- [Reproducibility notes](#reproducibility-notes)
- [License](#license)

## Repository layout

```text
.
├── README.md
├── LICENSE
├── data/                        # local inputs (data files are NOT stored in Git)
│   ├── cCRE/hg38/               # cell-type cCRE BEDs in hg38 (see its README for the source)
│   ├── gwas/                    # GWAS summary statistics (see its README for the sources)
│   └── ldsc_ref/                # LDSC reference files (see its README for the sources)
├── docs/
│   └── data-sources.md          # full data/software provenance, citations, and access notes
├── notebooks/
│   └── S_LDSC_notebook.qmd      # tidied research notes / protocol (Quarto)
└── workflow/
    ├── Snakefile                # the Snakemake pipeline
    ├── config.yaml              # cell types, traits, and file paths
    ├── profiles/default/        # Snakemake profile (defaults to 4 parallel jobs)
    ├── S_LDSC_notebook.qmd      # original analysis notebook; source of the data-download notes below
    └── cCRE/hg19/               # generated (lifted) cCRE files; not in Git
```

## Requirements

- Git and Conda (or Mamba)
- [Snakemake](https://snakemake.readthedocs.io/)
- LDSC in a Python 3.9 environment — the `ldsc39` branch of CBIIT/ldsc, set up in Step 1
- `bedtools`, `pybedtools`, and UCSC `liftOver` — installed in Step 1

## Getting started

The workflow is run from the `workflow/` directory, but the steps below start at the repository root. In order: set up LDSC, download the reference files, provide the input data, edit the config, run Snakemake.

### Step 1 — Clone and set up LDSC (Python 3.9 branch)

The analysis uses the Python 3.9 branch (`ldsc39`) of the CBIIT LDSC repository:

```bash
git clone -b ldsc39 https://github.com/CBIIT/ldsc.git
```

Create a Python 3.9 Conda environment and install LDSC's dependencies inside it:

```bash
conda create --name ldsc39 python=3.9
conda activate ldsc39
cd ldsc
pip install -r requirements.txt
```

> **Conda channel note.** If collecting package metadata fails with
> `UnavailableInvalidChannel: HTTP 403 FORBIDDEN` on the default channel,
> switch to conda-forge:
>
> ```bash
> conda config --remove channels defaults
> conda config --add channels conda-forge
> ```

Install the tools needed by the annotation and LiftOver steps:

```bash
conda install -c bioconda -c conda-forge pybedtools bedtools
conda install -c bioconda ucsc-liftover
```

Apply the small compatibility fix to `ldscore/parse.py` (needed because the workflow calls `--overlap-annot` with multiple comma-separated reference prefixes). In the `annot(fh_list, ...)` function, the chromosome lookup must use `fh_list[0]` instead of `fh`:

```python
chrs = get_present_chrs(fh_list[0], num + 1)
```

Record the commit you use (`git -C ldsc rev-parse HEAD`) and keep the patch — see [`docs/data-sources.md`](docs/data-sources.md) for details.

Finally, point `ldsc_dir` in [`workflow/config.yaml`](workflow/config.yaml) at this checkout (the default is `../../ldsc` relative to `workflow/`).

### Step 2 — Download the LDSC reference files

S-LDSC needs the 1000 Genomes EUR Phase 3 reference data. The files used here were downloaded from the sources below. **All of these references are in hg19/GRCh37 coordinates** (the reason cCREs are lifted from hg38 to hg19 later in the workflow).

| Download | Used for | Downloaded from |
|---|---|---|
| `1000G_Phase3_plinkfiles.tgz` | 1000G EUR PLINK genotypes (`1000G.EUR.QC.{1..22}.{bed,bim,fam}`) | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `1000G_Phase3_weights_hm3_no_MHC.tgz` | LDSC regression weights | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `1000G_Phase3_baseline_v1.2_ldscores.tgz` | baseline v1.2 LD scores | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `1000G_Phase3_baselineLD_v2.2_ldscores.tgz` | baseline-LD v2.2 LD scores | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `1000G_Phase3_frq.tgz` | allele-frequency files | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) |
| `w_hm3.snplist.gz` (→ `w_hm3.snplist`) | HapMap3 SNP list for munging and LD scores | [Zenodo 7773502](https://zenodo.org/records/7773502) |
| `eur_w_ld_chr.tar.gz` (→ `eur_w_ld_chr/`) | non-partitioned LD scores for `--rg` genetic correlation | [Zenodo 8182036](https://zenodo.org/records/8182036) |
| `hg38ToHg19.over.chain.gz` | UCSC LiftOver chain for the cCREs | [UCSC hgdownload](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/liftOver/hg38ToHg19.over.chain.gz) |

Notes (recorded in the analysis notebook):

- Zenodo [7768714](https://doi.org/10.5281/zenodo.7768714) is the "S-LDSC reference files" bundle (Steven Gazal). Zenodo [7773502](https://zenodo.org/records/7773502) and [8182036](https://zenodo.org/records/8182036) are free mirrors of the Alkes Group's `w_hm3.snplist.bz2` and `eur_w_ld_chr.tar.bz2`.
- The original Broad directory (<https://data.broadinstitute.org/alkesgroup/LDSCORE>) is now available on a requester-pays basis only (Google Cloud bucket `broad-alkesgroup-public-requester-pays`, $0.01–$0.15 per GB). Prefer the free Zenodo copies above.
- `baselineLF_v2.2.UKB.tar.gz` is free via the AWS Open Data Program (<https://broad-alkesgroup-ukbb-ld.s3.amazonaws.com/UKBB_LD/baselineLF_v2.2.UKB.tar.gz>); it is not used by this workflow.

Unpack the references into `data/ldsc_ref/` (same commands as the notebook):

```bash
cd data/ldsc_ref

mkdir -p 1000G_EUR_Phase3_plink
tar -xzf 1000G_Phase3_plinkfiles.tgz -C 1000G_EUR_Phase3_plink --strip-components=1
mkdir -p weights_hm3_no_MHC
tar -xzf 1000G_Phase3_weights_hm3_no_MHC.tgz -C weights_hm3_no_MHC --strip-components=1
mkdir -p baseline_v1.2
tar -xzf 1000G_Phase3_baseline_v1.2_ldscores.tgz -C baseline_v1.2 --strip-components=1
mkdir -p baselineLD_v2.2
tar -xzf 1000G_Phase3_baselineLD_v2.2_ldscores.tgz -C baselineLD_v2.2 --strip-components=1
mkdir -p 1000G_Phase3_frq
tar -xzf 1000G_Phase3_frq.tgz -C 1000G_Phase3_frq --strip-components=1
tar -xzf eur_w_ld_chr.tar.gz
gunzip w_hm3.snplist.gz
```

Put `hg38ToHg19.over.chain.gz` in the same directory. The exact layout the workflow expects is described in [`data/ldsc_ref/README.md`](data/ldsc_ref/README.md).

### Step 3 — Provide the input GWAS and cCRE files

None of the input data are stored in Git. Place them under `data/` as described below (and in the README of each subdirectory).

**GWAS summary statistics** go under `data/gwas/` — **hg19/GRCh37** coordinates, matching the LDSC references. Three input formats are supported (declared per trait in the config):

- `daner` — standard PGC daner files with per-SNP `FRQ_A`/`FRQ_U` and N columns (e.g. OCD, SCZ). The workflow computes per-SNP `Neff` and `MAF` automatically.
- `simple` — minimal fixed-N format (e.g. TS); case/control counts are set in the config.
- `premunged` — already-munged LDSC-compatible sumstats (e.g. ADHD, Anorexia); used as-is, no munging.

Sources used in this analysis:

| Trait | Example file | Format | Downloaded from |
|---|---|---|---|
| OCD | `daner_OCD_full_wo23andMe_190522.gz` | daner | PGC download page (<https://pgc.unc.edu/for-researchers/download-results/>): meta-analysis **excluding 23andMe** from the study [*Genome-wide analyses identify 30 loci associated with obsessive–compulsive disorder*](https://www.nature.com/articles/s41588-025-02189-z) |
| SCZ | `daner_PGC_SCZ52_0513a.hq2.gz` | daner | PGC 2014, [*Biological Insights From 108 Schizophrenia-Associated Genetic Loci*](https://doi.org/10.1038/nature13595) ([figshare scz2014](https://figshare.com/articles/dataset/scz2014/14672163)) |
| TS | `TS_Oct2018.gz` | simple | Yu et al. 2019 ([DOI 10.1176/appi.ajp.2018.18070857](https://doi.org/10.1176/appi.ajp.2018.18070857)), [figshare ts2019](https://figshare.com/articles/dataset/ts2019/14672232) |
| ADHD | `PASS.Attention_Deficit_Hyperactivity_Disorder_ADHD.Demontis2023.sumstats.gz` | premunged | already-munged sumstats (Demontis 2023) |
| Anorexia | `PASS.Anorexia_Nervosa.Watson2019.sumstats.gz` | premunged | already-munged sumstats (Watson 2019) |

The `PASS.*` already-munged files (including the additional `rg_comparison_traits`) come from the external collections referenced in the config (Price-lab bundle / traits_indep107). Verify the access and redistribution terms of every study before downloading or sharing; record provenance in [`docs/data-sources.md`](docs/data-sources.md).

**Cell-type cCREs** go under `data/cCRE/hg38/` as one BED file per cell type:

```text
data/cCRE/hg38/<CELLTYPE>.bed.txt
```

These are human-brain cCREs called from snATAC-seq at cell-type (subclass) level, downloaded from **CATlas**: <https://decoder-genetics.wustl.edu/catlasv1/humanbrain/data/cCREs/subclass/> (study: [*A comparative atlas of single-cell chromatin accessibility in the human brain*](https://pmc.ncbi.nlm.nih.gov/articles/PMC10852054/); related code: [snATACutils](https://github.com/yal054/snATACutils), [SnapATAC](https://github.com/r3fang/SnapATAC)).

**Genome build:** the CATlas cCRE files are **hg38**. The workflow **lifts them over to hg19** (UCSC `liftOver` + the chain from Step 2) so they match the **hg19/GRCh37** LDSC references and GWAS coordinates. The workflow therefore expects hg38 cCRE inputs; the hg19 versions are generated outputs.

### Step 4 — Configure the cell types and trait(s) of interest

Edit [`workflow/config.yaml`](workflow/config.yaml):

- `celltypes:` — the cell types to analyze. Each name must match a `<CELLTYPE>.bed.txt` file in `data/cCRE/hg38/`.
- `traits:` — the GWAS trait(s) of interest. Each entry sets `format` (`daner` / `simple` / `premunged`) and the input filename; `simple` traits also need `n_cas` / `n_con`.
- Genetic-correlation settings (optional): `rg_base_trait`, `rg_traits`, `rg_comparison_traits`, and the optional `exclude_snps` list.
- `ldsc_dir` and the input paths — adjust them if your files are not in the default locations.

For machine-specific path overrides that should not be committed, create the git-ignored `workflow/config.local.yaml` and add `--configfile config.local.yaml` to the Snakemake commands.

### Step 5 — Run the Snakemake workflow

From the `workflow/` directory, first inspect the DAG:

```bash
cd workflow
snakemake --profile profiles/default --dry-run
```

If the proposed jobs look right, run the workflow:

```bash
snakemake --profile profiles/default
```

**Parallel cores:** Snakemake runs independent jobs in parallel. The bundled profile ([`workflow/profiles/default/config.yaml`](workflow/profiles/default/config.yaml)) **defaults to 4 parallel jobs** (`jobs: 4`, the same as the `snakemake -j 4` used in the original analysis). Adjust `jobs` in the profile — or pass `-j N` — to match the CPU and memory available.

To build a single target instead of everything:

```bash
snakemake --profile profiles/default results/OCD/summary_table_corrected.tsv
```

After changing inputs or configuration, run the same command again: Snakemake rebuilds only missing or outdated outputs (and their dependants). Trait-independent annotation and LD-score work is computed once and reused across traits.

## Workflow stages

| # | Stage | What Snakemake does | Main outputs |
|---|---|---|---|
| 1 | GWAS preparation and munging | Prepares and munges raw GWAS files; passes already-munged files through unchanged. | `workflow/gwas/` |
| 2 | LiftOver | Converts each cCRE BED from hg38 to hg19 and sorts the mapped intervals. | `workflow/cCRE/hg19/` |
| 3 | Annotation | Builds chromosome-specific annotations for each cell type. | `workflow/annot/` |
| 4 | LD scores | Calculates chromosome-specific, HapMap3-restricted LD scores for each cell type (shared across traits). | `workflow/annot/` |
| 5 | Heritability | Runs partitioned h² for every configured trait × cell type against baseline v1.2 and baseline-LD v2.2. | `workflow/results/<trait>/` |
| 6 | Summaries | Aggregates cell-type results and applies FDR and Bonferroni correction. | `summary_table.tsv`, `summary_table_corrected.tsv` per trait |
| 7 | Genetic correlation | Runs configured pairwise trait comparisons and base-trait vs. external comparisons. | `workflow/results/rg/` |

With the checked-in config, a full run requests **430 h² regressions** (five traits × 43 unique cell types × two baseline models; the duplicate `PVALB` entry is deduplicated) and **24 unique rg comparisons**. Inspect the dry-run DAG and consider targeting a subset first.

## Outputs

Final outputs are written under `workflow/results/`:

- Per-trait, per-cell-type LDSC `.results` files for both baseline models.
- Per-trait `summary_table.tsv` and `summary_table_corrected.tsv` (coefficient tests with Benjamini–Hochberg FDR and Bonferroni correction, per baseline model).
- Per-pair rg logs and `rg_summary.tsv` under `workflow/results/rg/`.

Intermediates (munged GWAS, lifted cCREs, annotations, LD scores, logs) are written under `workflow/gwas/`, `workflow/cCRE/hg19/`, `workflow/annot/`, and `workflow/logs/`. All generated files are ignored by Git. Removing a trait or cell type from the config does not delete its existing outputs.

## Data sources and provenance

Where everything comes from (full details, citations, and access notes in [`docs/data-sources.md`](docs/data-sources.md)):

| Data | Genome build | Source | Also see |
|---|---|---|---|
| LDSC references (PLINK, weights, baseline v1.2, baseline-LD v2.2, frequencies) | hg19/GRCh37 | [Zenodo 7768714](https://doi.org/10.5281/zenodo.7768714) (S-LDSC reference files) | [`data/ldsc_ref/README.md`](data/ldsc_ref/README.md) |
| HapMap3 SNP list (`w_hm3.snplist`) | hg19/GRCh37 | [Zenodo 7773502](https://zenodo.org/records/7773502) | [`data/ldsc_ref/README.md`](data/ldsc_ref/README.md) |
| EUR LD scores for rg (`eur_w_ld_chr/`) | hg19/GRCh37 | [Zenodo 8182036](https://zenodo.org/records/8182036) | [`data/ldsc_ref/README.md`](data/ldsc_ref/README.md) |
| UCSC LiftOver chain | hg38 → hg19 | [UCSC hgdownload](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/liftOver/hg38ToHg19.over.chain.gz) | [`data/ldsc_ref/README.md`](data/ldsc_ref/README.md) |
| GWAS sumstats (OCD, SCZ, TS, ADHD, Anorexia, rg comparators) | hg19/GRCh37 | PGC download page, figshare, already-munged collections (see Step 3) | [`data/gwas/README.md`](data/gwas/README.md) |
| Brain cell-type cCREs (snATAC-seq, CATlas) | **hg38** (lifted to hg19 by the workflow) | [CATlas human-brain subclass cCREs](https://decoder-genetics.wustl.edu/catlasv1/humanbrain/data/cCREs/subclass/) | [`data/cCRE/hg38/README.md`](data/cCRE/hg38/README.md) |

The original analysis notebook is [`workflow/S_LDSC_notebook.qmd`](workflow/S_LDSC_notebook.qmd) (the source of the download notes above); a tidied protocol version lives at [`notebooks/S_LDSC_notebook.qmd`](notebooks/S_LDSC_notebook.qmd). If Quarto is installed, render either with `quarto render <notebook>.qmd`. The Snakemake workflow is the source of truth for production runs.

## Reproducibility notes

- Record the exact LDSC commit (`git -C ldsc rev-parse HEAD`) and retain the `ldscore/parse.py` patch described in Step 1 and [`docs/data-sources.md`](docs/data-sources.md).
- The workflow has no per-rule Conda environments yet; install the requirements in the active environment (Step 1). The notebook records Python 3.9 + the checkout's `requirements.txt`, but not a full environment lock.
- Reference files are large and are not downloaded automatically; keep them in approved storage and record source, version, checksum, and citation in [`docs/data-sources.md`](docs/data-sources.md).
- The tracked `workflow/gwas/`, `workflow/cCRE/hg19/`, `workflow/annot/`, and `workflow/ldsc_ref/` paths hold generated intermediates, not raw input storage.

## License

The code in this repository is MIT-licensed (see [`LICENSE`](LICENSE)). The license does not grant rights to redistribute the datasets or third-party software (LDSC, reference panels, GWAS sumstats, cCREs); check each provider's terms before downloading or sharing them.
