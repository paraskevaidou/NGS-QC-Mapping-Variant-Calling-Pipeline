# NGS-QC-Mapping-Variant-Calling-Pipeline
End-to-end NGS pipeline for quality control, read trimming, BWA mapping, BAM processing, and biallelic SNP calling using FreeBayes on HPC systems.

# Requirements
HPC modules used:
- sra-tools
- fastqc
- trimmomatic
- bwa
- samtools
- vcftools
- miniconda3 (for FreeBayes)

# Step-by-Step Commands (Single Sample Example)
Replace <...> with your filenames.

# 0) Download FASTQ from SRA (optional)
module load gcc/14.2.0 sra-tools/3.0.3

fastq-dump --split-files <SRA_ACCESSION>

Output:
- <SRA_ACCESSION>_1.fastq
- <SRA_ACCESSION>_2.fastq

# 1) Quality Control (FastQC) — Raw Reads
module load gcc/14.2.0 fastqc/0.12.1

fastqc <reads_R1.fastq> <reads_R2.fastq>

- Inspect:
- Per base sequence quality
- Adapter content
- Sequence duplication

# 2) Trimming (Trimmomatic PE)
Produces:
- Paired reads (*_P.fastq) → used for mapping
- Unpaired reads (*_U.fastq) → optional

module load gcc/9.4.0-eewq4j6 trimmomatic/0.39-dlgljoz

trimmomatic PE <reads_R1.fastq> <reads_R2.fastq> <sample>_R1_P.fastq <sample>_R1_U.fastq <sample>_R2_P.fastq <sample>_R2_U.fastq ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 LEADING:5 TRAILING:5 SLIDINGWINDOW:3:15 MINLEN:100

 Explanation of filters:
- ILLUMINACLIP → remove adapters
- LEADING/TRAILING → remove low-quality ends
- SLIDINGWINDOW → dynamic quality trimming
- MINLEN → discard short reads

# 3) FastQC Again (Trimmed Reads)
fastqc <sample>_R1_P.fastq <sample>_R2_P.fastq

Confirm:
- Adapter removal successful
- Improved quality profiles

# 4) Index Reference Genome (BWA)
module load gcc/14.2.0 bwa/0.7.17

bwa index <reference.fasta>

- This generates index files required for mapping.

# 5) Mapping (BWA-MEM via Slurm)
Create a file:

vim map.sbatch

Insert:
#!/bin/bash

module load gcc/14.2.0 bwa/0.7.17

bwa mem -t $SLURM_CPUS_PER_TASK <reference.fasta> <sample>_R1_P.fastq <sample>_R2_P.fastq > <sample>.sam

Run:
sbatch map.sbatch

Output:
<sample>.sam

# 6) SAM → BAM + Filtering + Sorting + Indexing
module load gcc/14.2.0 samtools/1.19.2

- Convert SAM to BAM
samtools view -h -b <sample>.sam > <sample>.bam

- Keep properly paired reads with MAPQ ≥ 20
samtools view -f 0x02 -q 20 -b <sample>.bam > <sample>_pp_mq20.bam

- Sort BAM
samtools sort <sample>_pp_mq20.bam -o <sample>_pp_mq20_sorted.bam

- Index BAM (required for variant calling)
samtools index <sample>_pp_mq20_sorted.bam

- Alignment statistics
samtools flagstat <sample>.bam
samtools flagstat <sample>_pp_mq20_sorted.bam

# 7) Count Properly Paired Reads (MQ≥20)
samtools view -c -f 0x02 -q 20 <sample>.bam

This gives the number of high-confidence mapped read pairs.

# 8) Variant Calling (FreeBayes)
- Activate conda
module load gcc/14.2.0 miniconda3
source $CONDA_PROFILE/conda.sh
conda activate <your_env_name>

- Install once:
conda install -c bioconda freebayes

- Index Reference FASTA (Required)
module load gcc/14.2.0 samtools/1.19.2
samtools faidx <reference.fasta>

- Run FreeBayes
freebayes -f <reference.fasta> -b <sample>_pp_mq20_sorted.bam -v <sample>.vcf

- Output:
<sample>.vcf

# 9) Extract Biallelic SNPs Only (VCFtools)
module load vcftools

vcftools --vcf <sample>.vcf --min-alleles 2 --max-alleles 2 --remove-indels --recode --out <sample>_biallelic_snps

- Count Biallelic SNPs
grep -v '^#' <sample>_biallelic_snps.recode.vcf | wc -l

# Notes
- MAPQ ≥ 20 is a common confidence threshold.
- Biallelic SNP filtering is particularly useful for downstream population genetics analyses.
- For multi-sample studies, joint variant calling is recommended.
