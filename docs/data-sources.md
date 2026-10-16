# Data and software provenance

This inventory consolidates sources named in the uploaded Quarto analysis notes and `workflow/config.yaml`. It is provenance documentation—not authorization to download or redistribute data. Check each provider's current terms, access requirements, version, and citation before use. No source datasets are stored in this repository.

## Primary cCRE and GWAS inputs

| Input | Source recorded in the analysis notes | Notes |
|---|---|---|
| OCD summary statistics | Psychiatric Genomics Consortium (PGC) [download page](https://pgc.unc.edu/for-researchers/download-results/); study: [Genome-wide analyses identify 30 loci associated with obsessive-compulsive disorder](https://www.nature.com/articles/s41588-025-02189-z#data-availability) | The configured input is the meta-analysis excluding 23andMe: `daner_OCD_full_wo23andMe_190522.gz`. Confirm the exact release and its access terms. |
| Brain cCREs | [CATlas human-brain subclass cCRE BED files](https://decoder-genetics.wustl.edu/catlasv1/humanbrain/data/cCREs/subclass/); study: [A comparative atlas of single-cell chromatin accessibility in the human brain](https://pmc.ncbi.nlm.nih.gov/articles/PMC10852054/) | The workflow expects one hg38 BED file per cell type under `data/cCRE/hg38/`. The notebook also referenced [snATACutils](https://github.com/yal054/snATACutils) and [SnapATAC](https://github.com/r3fang/SnapATAC) as related analysis code. |
| hg38-to-hg19 chain | [UCSC liftOver chain download](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/liftOver/hg38ToHg19.over.chain.gz) | Required by the `liftover` rule. |

## LDSC software and reference data

- **LDSC code:** the analysis notes specify the Python 3.9 `ldsc39` branch of [CBIIT/ldsc](https://github.com/CBIIT/ldsc/tree/ldsc39). The notebook does not record the exact commit used. Record `git -C <ldsc-checkout> rev-parse HEAD` for the version actually run; a branch name alone is not reproducible.
- **Compatibility patch:** the workflow's `--overlap-annot` use with multiple comma-separated reference prefixes requires the fix noted in `workflow/Snakefile`. In `ldscore/parse.py`, within `annot(fh_list, ...)`, the chromosome lookup needs to use `fh_list[0]` rather than the undefined/wrong `fh` variable:

  ```python
  chrs = get_present_chrs(fh_list[0], num + 1)
  ```

  Apply only after confirming the source context in your checked-out revision, and retain a record of the patched commit/diff. The original notebook applied this change with a line-number-based `sed` command; avoid relying on line numbers across revisions.
- **Baseline and supporting reference files:** the notes identify the minimal LDSC data bundle at [Zenodo record 7768714](https://doi.org/10.5281/zenodo.7768714), the HapMap3 list at [Zenodo record 7773502](https://zenodo.org/records/7773502), and EUR LD scores for genetic correlation at [Zenodo record 8182036](https://zenodo.org/records/8182036). The local layout expected by the workflow is documented in [`data/ldsc_ref/README.md`](../data/ldsc_ref/README.md).
- The notebook also notes that some files from the original Broad LDSCORE storage moved to requester-pays access. The workflow expects locally available reference files and does not download them automatically.

## Other configured GWAS traits

The notebook records these source citations for comparison datasets:

- Tourette syndrome: Yu et al., *Interrogating the Genetic Determinants of Tourette's Syndrome and Other Tic Disorders Through Genome-Wide Association Studies*, DOI [10.1176/appi.ajp.2018.18070857](https://doi.org/10.1176/appi.ajp.2018.18070857), [figshare data](https://figshare.com/articles/dataset/ts2019/14672232?file=28169937).
- Schizophrenia (2014): *Biological Insights From 108 Schizophrenia-Associated Genetic Loci*, DOI [10.1038/nature13595](https://doi.org/10.1038/nature13595), [figshare data](https://figshare.com/articles/dataset/scz2014/14672163).

The config also names already-munged comparison files for ADHD (Demontis 2023), anorexia (Watson 2019), depression (Howard 2019), bipolar disorder (Mullins 2021), schizophrenia (Trubetskoy 2022), insomnia (Watanabe 2022), BMI (Yengo 2018), height (Yengo 2022), alcohol use (Sanchez-Roige 2018), cigarettes per day (Liu 2019), endometriosis (Rahmioglu 2023), lupus (Bentham 2015), physical activity (Wang 2022), reaction time (Davies 2018), and sleep duration (Dashti 2019). These labels and filenames are taken from the config; the uploaded notes do not provide complete citations or access links for every one. Add verified citations, source URLs, release identifiers, and checksums before publication or reuse.

## Future directions from the notebook (not current inputs)

- Fetal/adult chromatin-accessibility data: *A single-cell atlas of chromatin accessibility in the human genome*, DOI [10.1016/j.cell.2021.10.024](https://doi.org/10.1016/j.cell.2021.10.024), and the [CATLAS human-enhancer portal](https://catlas.org/humanenhancer/data/).
- Endometrial cCREs: the notes name *Dynamic chromatin remodeling in cycling human endometrium at single cell level* but do not record a DOI or download URL; identify and verify the exact data release before use.

These sources are not included in the current workflow configuration.
