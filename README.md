# PsychCellLDSC

A Snakemake workflow for cell-type partitioned heritability and genetic-correlation analyses with stratified LD Score Regression (S-LDSC). It uses CATlas brain cell-type cis-regulatory elements (cCREs), lifts annotations from hg38 to hg19, computes LD scores, and analyzes GWAS traits against baseline and baseline-LD references.

## Project contents

- **Workflow:** `workflow/Snakefile` and `workflow/config.yaml` are the executable pipeline and its shared configuration.
- **Analysis notebook:** [`notebooks/S_LDSC_notebook.qmd`](notebooks/S_LDSC_notebook.qmd) is the cleaned Quarto research notebook/protocol. It records the analysis rationale and manual checks; Snakemake is the source of truth for production runs.
- **Data provenance:** [`docs/data-sources.md`](docs/data-sources.md) collects citations, download sources, and dependency notes from the notebook and config.
- **Inputs and results:** large or access-controlled datasets are kept locally under `data/`, not committed. Generated artifacts are written under `workflow/` and ignored by Git.

No GWAS summary statistics, cCRE BEDs, reference genotypes/LD scores, or analysis results are included. Follow the access and redistribution terms for every source.

## Analyses

1. Prepare raw GWAS summary statistics, or accept already-munged inputs.
2. Lift CATlas cCRE BED files from hg38 to hg19 and create chromosome-specific annotations.
3. Calculate HapMap3-restricted LD scores once per cell type, shared across traits.
4. Run partitioned heritability against `baseline_v1.2` and `baselineLD_v2.2`; summarize coefficient tests with Benjamini–Hochberg FDR and Bonferroni correction.
5. Run pairwise rg analyses among `rg_traits` and the configured base-trait comparisons against already-munged files in `rg_comparison_traits`.

The current config contains five h2 traits and 43 unique cell types, producing 430 h2 regressions in a full run. It also configures 24 unique rg comparisons after deduplication. Inspect the DAG and consider targeting a subset before launching the full analysis.

## Repository layout

```text
.
├── README.md
├── LICENSE
├── docs/
│   └── data-sources.md
├── notebooks/
│   └── S_LDSC_notebook.qmd
├── data/                         # local inputs; contents ignored except READMEs
│   ├── cCRE/hg38/README.md
│   ├── gwas/README.md
│   └── ldsc_ref/README.md
└── workflow/
    ├── Snakefile
    ├── config.yaml               # shared example/project settings
    ├── profiles/default/config.yaml
    └── cCRE/hg19/README.md       # generated output location
```

## Data layout

The workflow is run with `workflow/` as its working directory, so the config's relative paths resolve from there. The current config expects:

- `data/cCRE/hg38/<CELLTYPE>.bed.txt`: one CATlas hg38 BED for each configured cell type.
- `data/ldsc_ref/hg38ToHg19.over.chain.gz`: UCSC hg38-to-hg19 chain file.
- `data/ldsc_ref/1000G_EUR_Phase3_plink/1000G.EUR.QC.{1..22}.{bed,bim,fam}`: 1000 Genomes EUR PLINK data.
- `data/ldsc_ref/w_hm3.snplist`: HapMap3 SNP list.
- `data/ldsc_ref/baseline_v1.2/`, `baselineLD_v2.2/`, `weights_hm3_no_MHC/`, `1000G_Phase3_frq/`, and `eur_w_ld_chr/`: LDSC baseline, weight, frequency, and non-partitioned LD-score references.
- Raw and already-munged GWAS inputs under `data/gwas/`, with filenames configured in `workflow/config.yaml`.
- A separate CBIIT LDSC checkout with `ldsc.py`, `munge_sumstats.py`, and `make_annot.py`; set `ldsc_dir` to its path. The notebook records the Python 3.9 `ldsc39` branch and a compatibility fix in `ldscore/parse.py`; record the exact commit actually used.

The detailed input layout is described in the README files under `data/`. Do not put datasets in Git or Git LFS by default. Keep files in approved storage and record source, release/version, checksum, and citation in `docs/data-sources.md`. The tracked `workflow/ldsc_ref/` and `workflow/gwas/` paths are for generated intermediates, not raw input storage.

## Software and reproducibility

The notebook records a CBIIT `ldsc39` checkout, Python 3.9, the checkout's `requirements.txt`, plus `pybedtools`, `bedtools`, and UCSC `liftOver`. The exact LDSC commit and a complete environment lock were not recorded. The workflow currently has no per-rule Conda environments, so `--use-conda` alone does not make it self-contained. Before publishing results, pin the actual LDSC commit and tested software environment, retain the `parse.py` patch/diff, and document reference-data versions and citations. See [`docs/data-sources.md`](docs/data-sources.md) for details.

## Configure and run

1. Install Snakemake and the Python/system dependencies compatible with your LDSC checkout. 
2. Place local inputs in `data/` and confirm paths in `workflow/config.yaml`, especially `ldsc_dir`.
3. From the repository root, inspect the DAG:

   ```bash
   cd workflow
   snakemake --profile profiles/default --dry-run
   ```

4. Run after reviewing the dry-run output:

   ```bash
   snakemake --profile profiles/default
   ```

For machine-specific path overrides that should not be committed, create the ignored `workflow/config.local.yaml` and add `--configfile config.local.yaml` to the Snakemake commands while in `workflow/`. The supplied profile limits concurrency to four jobs; adjust `jobs` in `workflow/profiles/default/config.yaml` to suit available CPU and memory. A single summary target can be requested instead of `rule all`, for example:

```bash
snakemake --profile profiles/default results/OCD/summary_table_corrected.tsv
```

Outputs are written under `workflow/results/`; logs and intermediates are under `workflow/logs/`, `workflow/annot/`, `workflow/cCRE/hg19/`, and `workflow/gwas/`. These paths are ignored by Git.

The notebook is a Quarto document with static code examples and notes, not an executable copy of the whole Snakemake DAG. If Quarto is installed, render it with `quarto render notebooks/S_LDSC_notebook.qmd`; generated HTML and render caches are ignored.

