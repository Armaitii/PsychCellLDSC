# PsychCellLDSC

(still under development)

A Snakemake workflow for cell-type partitioned heritability and genetic-correlation analyses with stratified LD Score Regression (S-LDSC). It uses cell-type cis-regulatory elements (cCREs), lifts annotations from hg38 to hg19 if needed, computes LD scores, and analyzes GWAS traits against baseline and baseline-LD references.

## 1. Inputs you provide

Place the input files (cCRE and GWAS data) in the local `data/` directories and set their locations and study settings in [`workflow/config.yaml`](workflow/config.yaml). Run Snakemake from `workflow/`; the checked-in config uses paths such as `../data/gwas/...` to refer to the project-level input folders. The data files themselves are not stored in this GitHub repository.

### GWAS summary statistics

The current configuration uses these input types:

- **Raw `daner` files:** For example OCD and schizophrenia (SCZ), under `data/gwas/OCD/` and `data/gwas/SCZ/`. The workflow derives effective sample size and MAF from the source columns, then munges the files.
- **Raw `simple` file:** For example Tourette syndrome (TS), under `data/gwas/TS/`. Its case and control sample counts are supplied in the config.
- **Already-munged files:** Available GWAS sumstats, ADHD and Anorexia for example. Put these under `data/gwas/external/` as configured; they bypass raw-file preparation and munging.

Check the `traits` and `rg_comparison_traits` sections for the exact filenames. Do not rename a file without updating the config. Verify the data-access and redistribution terms for every study before downloading or sharing it.

### Cell-type cCREs

Provide one cis regulatory element BED file for every configured cell type, pay attention to hg19/hg38 folder. This repository has used CATlas human brain snATAC-derived cCRE:

```text
data/cCRE/hg38/<CELLTYPE>.bed.txt
```

For example, the configured `MSN` input is `data/cCRE/hg38/MSN.bed.txt`. The cell-type list is in `workflow/config.yaml`.

### Reference data and software

The workflow also needs:

- The UCSC hg38-to-hg19 chain file (if the cCRE were hg38)
- 1000 Genomes EUR Phase 3 PLINK files for chromosomes 1–22.
- The HapMap3 SNP list, baseline v1.2 and baseline-LD v2.2 references, LD-score weights, allele-frequency files, and the EUR LD scores used for genetic correlation.
- A separate CBIIT LDSC checkout with `ldsc.py`, `munge_sumstats.py`, and `make_annot.py`, including the `parse.py` compatibility fix described in [`docs/data-sources.md`](docs/data-sources.md).

The expected reference-data layout is described in [`data/ldsc_ref/README.md`](data/ldsc_ref/README.md). The notebook records the sources and software setup used during the original analysis; it does not record a complete environment lock or the exact LDSC commit.

## 2. Workflow steps at a glance

| Step | What Snakemake does | Main outputs |
|---|---|---|
| 1. GWAS preparation and checkings | Prepares and munges raw GWAS files; passes already-munged files through unchanged. | `workflow/gwas/` |
| 2. LiftOver | Converts each cCRE BED from hg38 to hg19 and sorts the mapped intervals. | `workflow/cCRE/hg19/` |
| 3. Annotation | Builds chromosome-specific annotations for each cell type. | `workflow/annot/` |
| 4. LD scores | Calculates chromosome-specific, HapMap3-restricted LD scores for each cell type. These are shared across GWAS traits. | `workflow/annot/` |
| 5. Heritability | Runs partitioned h2 for every configured trait and cell type against baseline v1.2 and baseline-LD v2.2. | `workflow/results/<trait>/` |
| 6. Summaries | Aggregates cell-type results and applies FDR and Bonferroni correction. | `summary_table.tsv` and `summary_table_corrected.tsv` under each trait's results folder |
| 7. Genetic correlation | Runs configured pairwise trait comparisons and base-trait comparisons against external, already-munged sumstats. | `workflow/results/rg/` |

## 3. Outputs

The final outputs are written under `workflow/results/`:

- Per-trait, per-cell-type LDSC `.results` files for both baseline models.
- Per-trait `summary_table.tsv` and `summary_table_corrected.tsv` files.
- Per-pair rg logs and `rg_summary.tsv` under `workflow/results/rg/`.

Intermediate GWAS files, lifted cCREs, annotations, LD scores, and logs are written under `workflow/gwas/`, `workflow/cCRE/hg19/`, `workflow/annot/`, and `workflow/logs/`. These generated files are ignored by Git.

With the current config, a full run requests **430 h2 regressions** (five traits × 43 unique cell types × two baseline models) and **24 unique rg comparisons**. The duplicate `PVALB` entry in the config is deduplicated by the Snakefile.

## 4. What you need to do

Once the inputs are in place and the config paths are correct, you do **not** need to run each analysis command or pipeline stage manually. Run Snakemake; it builds the dependency graph and schedules the required steps. Each time you rerun it, Snakemake checks declared file inputs, rule code, and relevant parameters, then rebuilds missing or outdated outputs and their downstream dependants. Trait-independent cCRE annotation and LD-score work is reused across traits.

After changing files or configuration, run the command again. Files and settings represented in the workflow's rules can trigger automatic invalidation; if you replace a prefix-based reference file or edit the optional `exclude_snps` list at the same path, force the affected rule or remove its old output. Removing a trait or cell type from the config also does not delete its existing output files.

## 5. Running the workflow

Install the software listed above, place inputs in `data/`, and check `ldsc_dir` in `workflow/config.yaml`. Then, from the repository root:

```bash
cd workflow
snakemake --profile profiles/default --dry-run
```

Review the proposed jobs and missing inputs. If the DAG looks right, run:

```bash
snakemake --profile profiles/default
```

The profile limits concurrency to four jobs; adjust `jobs` in `workflow/profiles/default/config.yaml` for the memory and CPU available. To run only one target, provide it after the command, for example:

```bash
snakemake --profile profiles/default results/OCD/summary_table_corrected.tsv
```

For machine-specific path overrides that should not be committed, create the ignored `workflow/config.local.yaml` and add `--configfile config.local.yaml` to the commands while working in `workflow/`.

## 6. Details of each step

### 1. GWAS preparation and munging

The `prep_gwas` rule handles the input format declared for each trait:

- `daner`: calculates per-SNP effective sample size from the available Neff/N fields and computes MAF from case/control frequencies.
- `simple`: calculates a constant effective sample size from `n_cas` and `n_con` in the config.
- `premunged`: uses the supplied LDSC-compatible summary statistics directly.

The `munge_sumstats` rule uses the HapMap3 list and writes munged statistics under `workflow/gwas/`. The raw inputs stay under `data/gwas/`.

### 2. cCRE LiftOver and sorting

For each configured cell type, `liftover` maps the hg38 BED intervals to hg19 using the configured UCSC chain file. `sort_bed` orders the mapped intervals for annotation generation. The chromosome prefix is retained because the configured LDSC `make_annot.py` expects it.

### 3. Annotation generation

`make_annot` intersects each sorted cell-type BED with the chromosome-specific 1000 Genomes EUR BIM files. Annotation files are created once per cell type and chromosome, not once per GWAS trait.

### 4. LD-score calculation

`l2_score` calls LDSC with `--l2`, `--thin-annot`, and the single-column HapMap3 SNP list. It produces LD scores for each cell type and chromosome. These files are reused by all downstream heritability analyses.

### 5. Partitioned heritability

`h2_regression` runs `ldsc.py --h2` for every trait × cell type × baseline-model combination. It combines the cell-type reference with the selected baseline annotations and uses the configured weights and allele-frequency files. The workflow expects the LDSC `parse.py` compatibility fix for multiple reference prefixes when `--overlap-annot` is used; see [`docs/data-sources.md`](docs/data-sources.md).

### 6. Summaries and multiple-testing correction

`summarize` collects each cell type's LDSC result into a per-trait table. `significance_testing` derives two-sided coefficient p-values from the reported z-scores and applies Benjamini–Hochberg FDR and Bonferroni correction separately for each baseline model. These summary tables are the convenient final h2 outputs; the original `.results` files are retained as well.

### 7. Genetic correlation

The workflow uses the `rg_traits` list for pairwise comparisons and `rg_base_trait` with each entry in `rg_comparison_traits` for the configured external comparisons. Before LDSC `--rg`, `drop_incompatible_snps` optionally removes the same rsIDs from every input using `exclude_snps`; if that file is absent or empty, the sumstats are copied unchanged. Results are collected in `workflow/results/rg/rg_summary.tsv`.

## Repository organization and provenance

```text
.
├── data/                  # local inputs; data files ignored by Git
├── docs/data-sources.md   # sources, citations, and software notes
├── notebooks/             # Quarto research notebook/protocol
└── workflow/              # Snakefile, config, profiles, generated outputs
```

The notebook is [`notebooks/S_LDSC_notebook.qmd`](notebooks/S_LDSC_notebook.qmd). It contains the analysis rationale and historical QC notes; the Snakemake workflow is the source of truth for production runs. It is a Quarto document with static code examples; if Quarto is installed, render it with `quarto render notebooks/S_LDSC_notebook.qmd`. For data citations, download links, and LDSC version notes, see [`docs/data-sources.md`](docs/data-sources.md).

Raw GWAS, cCRE, genotype, chain, and LD-score data are not included in Git. Check their terms and keep them in approved storage; the code's MIT license does not grant rights to redistribute the datasets or third-party software. The workflow currently does not provide per-rule Conda environments or a pinned LDSC revision, so record the exact LDSC commit and a tested software environment before treating the analysis as fully reproducible.
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

The notebook records a CBIIT `ldsc39` checkout, Python 3.9, the checkout's `requirements.txt`, plus `pybedtools`, `bedtools`, and UCSC `liftOver`. The workflow currently has no per-rule Conda environments, so `--use-conda` alone does not make it self-contained. Before publishing results, pin the actual LDSC commit and tested software environment, retain the `parse.py` patch/diff, and document reference-data versions and citations. See [`docs/data-sources.md`](docs/data-sources.md) for details.

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

