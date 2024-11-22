# Functional Analysis with HUMAnN 3.0
HUMAnN (HMP Unified Metabolic Analysis Network) is a computational tool designed for precise and efficient profiling of microbial metabolic pathways and molecular functions using metagenomic or metatranscriptomic sequencing data. It is versatile and can be applied to analyze any microbial community, not limited to the human microbiome.
Please refer to the original [HUMAnN](https://huttenhower.sph.harvard.edu/humann) page for more information. 

![HUMAnN Workflow](https://github.com/SusanTian/DAWG_Fall2024/blob/main/humann_workflow.png)

# HUMAnN 3.0 Installation Guide

```
# Create a new conda environment for the installation
conda create --name biobakery3 python=3.7
conda activate biobakery3
```

## Set conda channel priority:
```
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --add channels biobakery
```

## Install HUMAnN 3.0 software with demo databases (20min):
```
conda install humann -c biobakery
```

## Test the installation by run HUMAnN unit tests (~1 minute):
```
humann_test 
```

## Set Path for commands
```
WORKDIR=`pwd`
echo $WORKDIR
```

## conda install -c bioconda metaphlan
HUMAnN provides organism-specific community functional profiles, and to do so it first detects the organisms present in a community using MetaPhlAn, which makes MetaPhlAn a pre-requisite for HUMAnN
```
mkdir -p $WORKDIR/data/humann/dbs
humann_databases --download chocophlan DEMO $WORKDIR/data/humann/dbs
humann_databases --download uniref DEMO_diamond $WORKDIR/data/humann/dbs
```

# Downloading demo files
HUMAnN supports various types of input data, each available in multiple formats:
1. Quality-controlled shotgun sequencing reads  
   - The most common starting point  
   - Includes data from a metagenome (DNA reads) or metatranscriptome (RNA reads)  
   - Supported formats: fastq, fastq.gz, fasta, fasta.gz  
2. Pre-computed mappings of reads to database sequences  
   - Supported formats: sam, bam, blastm8  
3. Pre-computed (typically gene) abundance tables  
   - Supported formats: tsv, biom
```
mkdir -p $WORKDIR/data/humann/input
wget -P $WORKDIR/data/humann/input https://github.com/biobakery/humann/raw/master/examples/demo.fastq.gz               
wget -P $WORKDIR/data/humann/input https://github.com/biobakery/humann/raw/master/examples/demo.sam               
wget -P $WORKDIR/data/humann/input https://github.com/biobakery/humann/raw/master/examples/demo.m8
```

## Check the downloaded files
```
ls -lh $WORKDIR/data/humann/input
```

## Process each file with HUMAnN 
```
humann -i $WORKDIR/data/humann/demo.fastq.gz -o $WORKDIR/data/humann/outputs/demo_output
```

## OR, process all samples at once if you have multiple files
```
for f in $WORKDIR/data/humann/*.fasta; do humann -i $f -o $WORKDIR/data/humann/output; done
```


# Post-processing
## Inspect the files
This file contains the abundances of each gene family in the community in reads per kilobase (RPK) units. Each gene family's total abundance is broken down into the contributions from individual organisms when possible (delineated by the pipe character). For example:
-less: A pager utility that allows you to view the contents of a file or output one screen at a time.
```
less $WORKDIR/data/humann/output/demo_genefamilies.tsv
```
the format is really complicated to see, so let's break it apart
-column: A command that formats text input into columns.
-t: Specifies that the output should be aligned into a table, ensuring that the columns are neatly spaced.
-s $'\t': Indicates that the input data is tab-separated (\t is the tab character), and the command should use this as the delimiter between columns.
-less: A pager utility that allows you to view the contents of a file or output one screen at a time.
-S: Prevents line wrapping (long lines that exceed the width of the terminal will remain on one line, and you can scroll horizontally to view them).
```
column -t -s $'\t' $WORKDIR/data/humann/output/demo_genefamilies.tsv | less -S
```

## Normalize gene abundance by gene length (copies per million)
HUMAnN natively normalizes gene family abundance by gene length. Why is this step important? (In other words, why are RPK units more interpretable than raw read counts?)
```
humann_renorm_table -i $WORKDIR/data/humann/output/demo_genefamilies.tsv -o $WORKDIR/data/humann/outputs/genefamilies_cpm.tsv --units cpm
```

## What are the unique gene families?
```
cut -f1 $WORKDIR/data/humann/output/demo_genefamilies.tsv | tail -n +3 | grep -v "|" | sort -u 
```
-cut -f1: Extracts the first column (-f1)
-tail -n +3: Skips the first two lines of the output and starts displaying from the third line onward.
-grep -v "|": Filters out lines containing the | character, -v specifies inversion, meaning it excludes lines with |
-sort -u: Sorts the remaining lines alphabetically and removes duplicates (-u for unique).
-wc -l: can be added to count how many unique gene families are


## Next, regroup functions to MetaCyc reactions (RXN)
Let's regroup our CPM-normalized gene family abundance values to MetaCyc reaction (RXN) abundances to enable functional interpretation by linking gene activity to specific biochemical reactions and pathways
```
humann_regroup_table --input $WORKDIR/data/humann/output/genefamilies_cpm.tsv \
    --output $WORKDIR/data/humann/output/rxn_cpm.tsv --groups uniref90_rxn

less $WORKDIR/data/humann/output/rxn_cpm.tsv
```

## Attaching names to features
We've been working with UniRef90 and RXN codes, which are efficient for HUMAnN but attaching human-readable descriptions to these IDs helps with biological interpretation.
```
humann_rename_table --input $WORKDIR/data/humann/output/rxn_cpm.tsv \
    --output $WORKDIR/data/humann/output/rxn-cpm-named.tsv --names metacyc-rxn

less $WORKDIR/data/humann/output/rxn-cpm-named.tsv
```
