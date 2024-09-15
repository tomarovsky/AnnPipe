# Required Software (might be incomplete)
|     Tool/Package     | version | Source |                  Reason                  | 
|:--------------------:|:-------:|:------:|:----------------------------------------:|
|        python        |  =3.9   | conda  |     dependency of make_lastz_chains      |
|       nextflow       |         | conda  |     dependency of make_lastz_chains      |
|     twobitreader     |         | conda  |     dependency of make_lastz_chains      |
|        lastz         |         | conda  |     dependency of make_lastz_chains      |
|    ucsc-axtchain     |         | conda  |     dependency of make_lastz_chains      |
|   ucsc-twoBitToFa    |         | conda  |     dependency of make_lastz_chains      |
|   ucsc-faToTwoBit    |         | conda  |     dependency of make_lastz_chains      |
| ucsc-chainAntiRepeat |         | conda  |     dependency of make_lastz_chains      |
| ucsc-chainMergeSort  |         | conda  |     dependency of make_lastz_chains      |
|    ucsc-chainSort    |         | conda  |     dependency of make_lastz_chains      |
|    ucsc-chainNet     |         | conda  |     dependency of make_lastz_chains      |
|    ucsc-axtToPsl     |         | conda  |     dependency of make_lastz_chains      |
|   ucsc-chainFilter   |         | conda  |     dependency of make_lastz_chains      |
|        py_nf         |         |  pip   |     dependency of make_lastz_chains      |
|      RouToolPa       |         | conda  |            dependency of MAVR            |
|         MAVR         |         | conda  | conversion of ncbi gff to bed12 for TOGA |
|  make_lastz_chains   |         |  git   |            generator of chain            |
|         TOGA         |         |  git   |           annotation transfer            |
|       postTOGA       |         |  git   |  postprocessing TOGA output (optional)   |
|       bed2gff        |         |  git   |        conversion of TOGA output         |
|       gffread        |         | conda  |        sorting resulting gff file        |

Conda channels: bioconda and mahajrod 

# Pipeline
I. download reference genome and corresponding annotation.
II. mask repeats in both query and reference genomes. This step is OBLIGATORY!
III. remove versions from scaffold ids in both query and reference fasta files. **make_lastz_chains** uses a lot of old UCSC software which handles versions incorrectly.
```commandline
trim_scaffold_versions.py -i <input_fasta> -o <no_versions_fasta>
```
IV. Convert both query and reference fasta to 2bit files. Both **make_lastz_chains** and **TOGA** use 2bit files as input
```commandline
faToTwoBit <no_versions_fasta> <no_versions_2bit>
```
V. convert ncbi gff to bed12 with . Necessary for TOGA.
```commandline
convert_ncbi_gff_to_bed12.py -i <reference_gff> -o <output_prefix> -c
```
-c option is necessary to remove scaffold vertions from gff, like it was done for fast files.
Output contains two important files:

<output_prefix>.final.mRNA.withCDS.bed
<output_prefix>.mRNA.isoforms.tab

VI. run make_lastz_chains
```commandline
<path_to_make_lastz_chains_dir>/make_chains.py gasterosteus_aculeatus_aculeatus SpiSpi1.hap1 gasterosteus_aculeatus_aculeatus.no_versions.2bit SpiSpi1.draft_qc.hap1.2bit --executor local --project_dir gasterosteus_aculeatus_aculeatus.to.SpiSpi1.hap1 --seq1_chunk 50000000 --seq2_chunk 50000000
```

VII. run TOGA
```commandline
<path_to_toga>TOGA/toga.py <chain_file> <bed12_with_annotations> <reference.2bit> <query.2bit> --project_dir <project_dir_name> --project_name <project_name> --kt  --isoforms <isoform.file> 2>&1 | tee toga.log.t
```

VIII. postprocess TOGA 
```commandline

tail -n +2 query_isoforms.tsv | cut -f 2 > query_isoforms.transcripts.tsv
filter_query_annotation_by_transcript_ids.py -i query_annotation.bed -d query_isoforms.transcripts.tsv  -o query_annotation.filtered.bed
restore_genes_for_isoforms.py -i query_isoforms.transcripts.tsv -r /maps/projects/codon_0000/people/svc-rapunzl-smrtanl/yggdrasil/db/genomes/passeriformes/catharus_ustulatus/assemblies/GCF_009819885.2/catharus_ustulatus.swainsons_thrush.GCF_009819885.2.mRNA.isoforms.tab -o query_isoform.original_names.tsv

$SOFT/TOGA/modules/get_transcripts_quality.py  temp/exons_meta_data.tsv  orthology_scores.tsv  transcript_quality.tsv
filter_query_annotation_by_transcript_ids.py -i temp/transcript_quality.tsv  -f 0 -o transcript_quality.filtered.tsv -d query_isoforms.transcripts.tsv
```

IX. Convert bed12 to gff
```commandline
bed2gff --bed query_annotation.filtered.bed  -i query_isoform.original_names.tsv  -o query_annotation.original_names.filtered.gff
bed2gff --bed query_annotation.bed  -i query_annotation.isoforms.tsv  -o query_annotation.gff

```

X. Sort converted gff 
```commandline
PREFIX=query_annotation.original_names.filtered
gffread ${PREFIX}.gff -o ${PREFIX}.gffread.gff -x ${PREFIX}.gffread.CDS.fasta -y ${PREFIX}.gffread.protein.fasta  -w ${PREFIX}.gffread.transcripts.fasta -g <query_genome_fasta>

```

# Usage
Install Apptainer (formerly Singularity) and create a SIF image based on the DEF file from this repository:
```commandline
apptainer build annpipe.sif annpipe.def
```

All pipelines commands are now available to run:
```commandline
ANNPIPE="/path/to/annpipe.sif"
apptainer exec --bind $(pwd) --pwd $(pwd) $ANNPIPE <TOOL> ...
```



