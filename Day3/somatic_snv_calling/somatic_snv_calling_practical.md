---
title: "Somatic SNV Calling Practical"
author: "Matt Eldridge, Cancer Research UK Cambridge Institute"
date: "July 2017"
output:
  html_document:
    toc: yes
    toc_depth: 4
    toc_float: yes
---

### Introduction

In this practical you will be running CaVEMan to identify somatic single nucleotide variants (SNVs) in the HCC1143 breast cancer cell line using aligned sequence data from both the tumour cell line and a cell line dervied from a matched peripheral blood sample taken from the same patient (HCC1143_BL).

[CaVEMan](http://cancerit.github.io/CaVEMan) (**Ca**ncer **V**ariants through **E**xpectation **Ma**ximizatio*n*) is part of an integrated bioinformatics analysis pipeline developed at the Wellcome Trust Sanger Institute as part of their Cancer Genome Project (**CGP**). It has been used on a number of projects as part of the International Cancer Genome Consortium ([ICGC](http://icgc.org)).

CaVEMan applies an expectation maximisation algorithm to call single nucleotide substitutions. Through comparison of reads from both tumour and normal with the reference genome, CaVEMan calculates a probability for each possible genotype per base (given tumour and normal copy number). In order to provide more accurate estimates of sequence error rates within the algorithm, thus aid identification of true variants, variables such as base quality, read position, lane, and read orientation are incorporated into the calculations.


### Start up a Sanger CGP docker container

Double click on the `Sanger_Docker` desktop icon.

Alternatively, if you don't have this desktop icon, open a new terminal window and type `Sanger_Docker` at the command line.

This should open a terminal window with a command-line prompt something like the following.

```
ubuntu@7882e0f18d5f:~$
```

The Sanger CGP docker image packages a complete environment including all the tools to run the CGP analysis pipeline. These tools include:

* **CaVEMan** for identifying somatic SNVs
* **Pindel** for identifying somatic indels (short insertions and deletions)
* **ASCAT** for detecting copy number changes
* **BRASS** for identifying somatic genomic rearrangements, also known as structural variants (SVs)


### Check the data are accessible

The Sanger docker environment should have access to the BAM files and reference data but we'll check this before attempting to run CaVEMan.

Change directory to the directory containing the BAM files and view the header of BAM file for the tumour cell line by typing the following at the command-line prompt.

```
cd /data
samtools view -H HCC1143.bam
```

Now check that the reference data is also accessible.

```
cd /reference_data
cd core_ref_GRCh37d5
cat genome.fa.fai
```

The `cat` command lists the contents of the index for the reference genome sequence. There is a row for each chromosome or contig.


### Running CaVEMan

CaVEMan is executed in several distinct steps in the order given below. While running these steps it may be helpful to follow the CaVEMan [guide](http://cancerit.github.io/CaVEMan) in parallel. This provides some more information about each step and lists the available optional settings for each.

The Sanger CGP pipeline executes all the steps for CaVEMan in a single workflow, along with the tools for calling the other types of variants. In this practical we will run the separate steps manually for a region of chromosome 21.

#### Set up a CaVEMan run

Change directory to the working directory in which we'll run CaVEMan.

```
cd /caveman_practical
```

Run `caveman setup` to create the configuration files for running CaVEMan specifying the tumour and normal BAM files, the reference genome sequence and a list of regions to exclude from the analysis. In the following command we exclude a set of regions that are usually sequenced to very high depth and cause problems for algorithms trying to identify mutations.

Run the following command.

```
caveman setup \
  --tumour-bam /data/HCC1143.bam \
  --normal-bam /data/HCC1143_BL.bam \
  --reference-index /reference_data/core_ref_GRCh37d5/genome.fa.fai \
  --ignore-regions-file /reference_data/SNV_INDEL_ref/caveman/HiDepth.tsv \
  --tumour-copy-no-file tum.cn.bed \
  --normal-copy-no-file norm.cn.bed
```

This creates two configuration files, `caveman.cfg.ini` and `alg_bean`. Take a look at the contents of these files.

```
cat caveman.cfg.ini
cat alg_bean
```

Note that we provided as inputs to CaVEMan the BAM files for both tumour and normal, the reference genome index file, a list of regions to exclude from the analysis and copy number estimates for both tumour and normal. The copy number estimates were generated by ASCAT.

#### Split chromosome 21 into chunks

CaVEMan divides the computational work into a number of jobs to be run in parallel on a multi-processor computer, by splitting the genome into regions. The size of each region is determined based on these containing a configurable number of aligned reads each.

In this run we will just be looking at a region of chromosome 21. To split this chromosome into chunks type the following at the command prompt.

```
caveman split -i 21
```

This takes a couple of minutes to scan through the reads on chromosome 21 and creates a file called `splitList.21` which can be viewed as follows.

```
cat splitList.21
```

The Sanger CGP pipeline runs `caveman split` for all chromosomes and then concatenates these into a single file called `splitList`. For this practical we will just work on the first 3 chunks of chromosome 21, so we'll use the `head` command to take just the first 3 rows.

```
head -3 splitList.21 > splitList
```

#### Run the maximization step

Run the following to perform the maximization step (M-step) for the first chunk.

```
caveman mstep -i 1
```

It should take about half a minute for this one chunk. In total the genome will typically be split into a few thousand chunks so overall this is a computationally expensive step.

The outputs are covariates files within a results directory. These files are binary files, the contents of which cannot be easily viewed.

```
ls results/21
```

Now run `caveman mstep` for the second and third chunks.

```
caveman mstep -i 2
caveman mstep -i 3
```

#### Merge the covariate files

The next step is to merge the covariates files generated for every region. In a full pipeline run there would be one such file for each of a large number of regions. We just have the three regions for this cut-down example.

```
caveman merge
```

This should produce two files, `probs_arr` and `covs_arr`, in the working directory.

#### Run the expectation step

The expectation step (E-step) is the final step in calling variants using CaVEMan. Like the maximization step, this needs to be run as separate jobs, one for each chunk.

Run the expectation step for the first chunk.

```
caveman estep -i 1
```

This is a another computationally expensive step but for a single chunk it should take a couple of minutes.

This step produces variant files in the Variant Call Format (VCF) for both germline and somatic variants, ending in `.snps.vcf` and `.muts.vcf` respectively.

```
ls results/21
```

List the contents of the file `1_9673176.muts.vcf` file, scroll back through the file and look at the header lines, those staring with `#`. Match the `FORMAT` definitions with the values given for the first mutation listed.

```
cat results/21/1_9673176.muts.vcf
```

If you wish, run the expectation step for the other two regions.

```
caveman estep -i 2
caveman estep -i 3
```


### Questions

Locate the `vcfProcessLog` header line in the VCF file, `1_9673176.muts.vcf`.

* _What prior probabilities were used for the SNP rate and the somatic mutation rate?_

Find the mutation call at position `9413844` on chromosome 21 and try to understand the information provided about this SNV, referring to the `INFO` and `FORMAT` header lines.

* _What is the base substitution for this mutation?_

* _What is the number of sequence reads aligned at this position in the normal? What is the depth in the tumour?_

* _How many reads support the alternate allele in the tumour? And the normal?_

* _What is the proportion of the mutant allele in the tumour and in the normal?_

* _What does CaVEMan think is the most probable joint genotype? What is the second most probable genotype? What are the probabilities for each?_

* _Does this seem like a credible somatic SNV?_

<div style="line-height: 250%;"><br></div>

