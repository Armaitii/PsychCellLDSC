# GWAS inputs

Place raw and already-munged summary-statistics files under this directory, following the paths in `workflow/config.yaml` (including the `OCD`, `SCZ`, `TS`, and `external` subdirectories). Paths in that config are relative to the `workflow/` working directory. Do not rename inputs without updating the config.

Raw summary statistics and the optional `exclude_snps.txt` list are excluded from Git. Verify the access and redistribution terms for every study; do not commit controlled or otherwise restricted data. Record source, version, citation, and checksum information in `docs/data-sources.md` without copying restricted data into this repository. Derived munged and rg-filtered sumstats are written separately under `workflow/gwas/` and are also ignored.
