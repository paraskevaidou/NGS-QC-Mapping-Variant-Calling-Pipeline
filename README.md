# NGS QC, Mapping & Variant Calling Genomics Pipeline

This pipeline provides a complete workflow for analyzing Illumina sequencing data in the context of genomics and population genetics. 
It includes steps for quality assessment, adapter removal, alignment to a reference genome, and variant detection.
It is optimized for high-performance computing (HPC) environments that utilize module systems such as Slurm.

The workflow was developed based on teaching material by Prof. Spyridon Papakostas

---

# Table of Contents

- [Requirements](#requirements)
- [Pipeline Overview](#pipeline-overview)
- [Step-by-Step Workflow](#step-by-step-workflow)
  - [0. Download Data](#0-download-data-optional)
  - [1. Quality Control (Raw Reads)](#1-quality-control-raw-reads)
  - [2. Trimming](#2-trimming-trimmomatic)
  - [3. Quality Control (Trimmed Reads)](#3-quality-control-trimmed-reads)
  - [4. Reference Indexing](#4-reference-indexing)
  - [5. Mapping (BWA-MEM)](#5-mapping-bwa-mem)
  - [6. BAM Processing](#6-bam-processing)
  - [7. Read Count (QC Metric)](#7-read-count-qc-metric)
  - [8. Variant Calling (FreeBayes)](#8-variant-calling-freebayes)
  - [9. Extract Biallelic SNPs](#9-extract-biallelic-snps)
- [Notes](#notes)

---

# Requirements

The pipeline assumes an HPC environment with the following modules:

- ``` sra-tools ```

- ``` fastqc ```

- ``` trimmomatic ```

- ``` bwa ```

- ``` samtools ```

- ``` vcftools ```

- ``` miniconda3 ``` (for FreeBayes)


# Pipeline Overview
``` SRA → FASTQ → FastQC → Trimming → FastQC → BWA → BAM → Filtering → Sorting → Indexing → FreeBayes → VCF → SNP Filtering ```


## Step-by-Step Workflow

Replace ``` <...> ``` placeholders with your filenames.

### 0. Download Data (Optional)

```
module load gcc/14.2.0 sra-tools/3.0.3

fasterq-dump <accession_number> --split-files
```

Output:

``` <accession_number>_1.fastq ```

``` <accession_number>_2.fastq ```


### 1. Quality Control (Raw Reads)

```
module load gcc/14.2.0 fastqc/0.12.1

fastqc -o fastqc_raw/ <reads_R1.fastq> <reads_R2.fastq>
```

Inspect:

- Per base sequence quality

- Adapter content

- Sequence duplication


### 2. Trimming (Trimmomatic)

```
module load gcc/9.4.0 trimmomatic/0.39
```

```
trimmomatic PE -threads 8 \
<reads_R1.fastq> <reads_R2.fastq> \
<sample>_R1_P.fastq <sample>_R1_U.fastq \
<sample>_R2_P.fastq <sample>_R2_U.fastq \
ILLUMINACLIP:/path/to/TruSeq3-PE.fa:2:30:10 \
LEADING:5 TRAILING:5 SLIDINGWINDOW:3:15 MINLEN:100
```

Output:

Paired reads: ``` *_P.fastq ``` (used for mapping)

Unpaired reads: ``` *_U.fastq ``` (optional)


### 3. Quality Control (Trimmed Reads)

```
fastqc -o fastqc_trimmed/ <sample>_R1_P.fastq <sample>_R2_P.fastq
```

Confirm improved quality and adapter removal


### 4. Reference Indexing

```
module load gcc/14.2.0 bwa/0.7.17

bwa index <reference.fasta>
```

### 5. Mapping (BWA-MEM)

Create a Slurm script:

```
vim map.sbatch
#!/bin/bash
#SBATCH --cpus-per-task=8
```

```
module load gcc/14.2.0 bwa/0.7.17 samtools/1.19.2
```

```
bwa mem -t $SLURM_CPUS_PER_TASK \
<reference.fasta> \
<sample>_R1_P.fastq <sample>_R2_P.fastq | \
samtools view -b -o <sample>.bam
```

Submit job:

``` sbatch map.sbatch ```

Output:

``` <sample>.bam ```


### 6. BAM Processing

```
module load gcc/14.2.0 samtools/1.19.2
```

- Filter properly paired reads (MAPQ ≥ 20)

```
samtools view -f 0x02 -q 20 -b <sample>.bam > <sample>_pp_mq20.bam
```

- Sort BAM

```
samtools sort -@ 8 <sample>_pp_mq20.bam -o <sample>_pp_mq20_sorted.bam
```

- Index BAM

```
samtools index <sample>_pp_mq20_sorted.bam
```

- Alignment statistics

```
samtools flagstat <sample>.bam

samtools flagstat <sample>_pp_mq20_sorted.bam
```

### 7. Read Count (QC Metric)

```
samtools view -c -f 0x02 -q 20 <sample>.bam
```

Returns number of high-confidence mapped read pairs


### 8. Variant Calling (FreeBayes)

- Activate Conda

```
module load gcc/14.2.0 miniconda3
source $CONDA_PROFILE/conda.sh
conda activate <your_env_name>
```

- Install FreeBayes (one-time)

```
conda install -c bioconda freebayes
Index Reference
samtools faidx <reference.fasta>
```

- Run FreeBayes

```
freebayes -f <reference.fasta> -p 2 \
-b <sample>_pp_mq20_sorted.bam \
-v <sample>.vcf
```

Output:

``` <sample>.vcf ```


### 9. Extract Biallelic SNPs

```
module load vcftools
vcftools --vcf <sample>.vcf \
--min-alleles 2 --max-alleles 2 \
--remove-indels \
--recode --out <sample>_biallelic_snps
```

Count SNPs

```
grep -v '^#' <sample>_biallelic_snps.recode.vcf | wc -l
```

# Notes

- MAPQ ≥ 20 is a common threshold for high-confidence alignments

- Explicitly set ploidy (-p) in FreeBayes depending on organism

- Biallelic SNP filtering is useful for:

	- Population genetics

	- GWAS

- For multi-sample datasets:

	- Consider joint variant calling
