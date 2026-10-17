# Generated cCRE annotations (hg19)

Snakemake writes lifted and sorted BED files here: the `liftover` and `sort_bed` rules convert each hg38 cCRE input from [`data/cCRE/hg38/`](../../../data/cCRE/hg38/) (CATlas human-brain snATAC-seq cCREs) to **hg19** with the UCSC chain file from `data/ldsc_ref/hg38ToHg19.over.chain.gz`, because the LDSC references are hg19/GRCh37.

These files are generated outputs and are excluded from Git. Keep the source BED files and the chain file available locally to reproduce them; do not edit files here by hand.
