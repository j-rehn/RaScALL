
# RaScALL - Rapid (Ra) Screening (Sc) of RNA-seq data for genomic alterations in Acute Lymphoblastic Leukaemia (ALL)

## Overview

Jimmy Breen : Jimmy.Breen@telethonkids.org.com 
Jacqueline Rehn : jacqueline.rehn@sahmri.com

This approach to detection of clinically relevant variants in ALL utilises jellyfish (https://github.com/gmarcais/Jellyfish) and a k-mer variant detection software 'km' (https://github.com/iric-soft/km). Additionally, RaScALL provides targets designed for detection of ALL specific gene fusions and variants. Within this repository there are targets developed for detection of recurrently observed in-frame gene fusions as well as pathogenic SNVs, with additional targets for detection of intragenic deletions of _IKZF1_ and _ERG_. Targets have also been developed for detection of _IGH-EPOR_ and _IGH-CRLF2_ as well as detection of _DUX4_ expression indicative of a _DUX4_-rearrangement. 

Together these tools can rapidly detect the presence of many ALL specific genomic alterations from raw RNA-seq data (fastq). The instructions below will enable installation of both km & jellyfish as well as provide a simple script to perform variant detection from fastq files. Incorporation of R and samtools into the environment enables the user to generate their own custom target sequences for variant detection.


## Create conda environment

1. To install and run RaScALL we'll need to create a conda environment for installation of km, jellyfish and samtools. Jellyfish needs additional python bindings to work with km, so we have created an install procedure to follow. This procedure requires that the command-line utility 'curl' be installed on your machine.

```
conda create -n RaScALL python=3.7 virtualenv -y
conda activate RaScALL
conda config --add channels bioconda
conda config --add channels conda-forge
conda install samtools==1.11
```

2. Next we will clone the git repository containing the targets generated for detecting ALL specific fusions/variants as well as bash scripts to install and run km against these targets. 

```
git clone https://github.com/jimmybgammyknee/RaScALL.git
```

3. Now we can install the km/jellyfish components using the `install.sh` script. First we change into the RaScALL directory generated by git, or an alternate directory for the git-repository if one was spcified. We then run the install script km and jellyfish will be isntalled into this directory. Note the install script requires `curl` be installed on your machine. 

```
cd RaScALL
bash install_km_Release13.sh
```

4. Finally, we wish to install R into the conda environment to allow use of Rscripts to filter output from km and also generate custom target sequences. If you already have R installed and in the path then this step may not be required.

```
conda install -c conda-forge r-base r-essentials
```


## Quick Start

We now wish to test km against a single RNA-seq sample utilising the ALL_targets. 

ALL_targets have been subdivided according to the main mutational type the target aims to detect: Gene fusions, sequence variants (single/di-nucleotide variants or small indels), Focal deletions of _IKZF1/ERG_, _DUX4_ expression or _IGH_ rearrangements with _CRLF2/EPOR_.

The script run_km.sh requires as input the path to R1 and R2 fastq files for a sample, as well as an argument specifying the mutation type or set of targets to test against. This script will first use jellyfish to generate a k-mer count table (k=31b) from the raw fastq files. Secondly, a command is called to run the countTable31.jf against the specified set of targets.

1. To view the required arguments for run_km.sh:

```
bash run_km.sh

Usage: run_km.sh [FASTQ_R1] [FASTQ_R2] [THREADS] [TARGET_DIR]
Incorrect number of arguments
- Required: FASTQ_R1 = FASTQ PAIRED-END READ 1
- Required: FASTQ_R2 = FASTQ PAIRED-END READ 2
- Required: THREADS = Number of threads for running jellyfish
- Required: TARGET_DIR = Location of directory containing targets for testing
```

Full path to the fastq location needs to be provided. 

2. In the example below fastq files are run against ALL specific SNV targets, using 12 threads. Modify the command to specify the path to location of own fastq files as well as location of directory containing SNV targets. As jellyfish is an in-memory k-mer counting algorithm memory requirements for will depend on size of the fastq files. Increasing the number of available threads decreases the run-time. 12 threads can be used succesfully for smaller fastq files (e.g. 75b length 60M reads) but it is suggested this be reduced for much larger files at the expense of additional run-time.

```
bash run_km.sh ~/test_data/testSampleA_1.fastq.gz ~/test_data/testSampleA_2.fastq.gz 12 ~/km_ALL_targets/ALL_targets/SNV
```

This script generates a subdirectory within `RaScALL` called `output` for output files. Within this k-mer count tables from jellyfish and text files generated by km for a given sample/targetType will be written into a sample directory labelled with basename of the fastq files.

```
ls output/testSampleA/
countTable31.jf  testSampleA_SNV.txt
```

3. Separate .txt files are generated with output from each of the target directories (SNV.txt, Fusion.txt, focal_deletion.txt, DUX4.txt and IGH_fusion.txt). These additional outputs can be generated by altering the \[TARGET_TYPE] argument as follows:

```
bash run_km.sh ~/test_data/testSampleA_1.fastq.gz ~/test_data/testSampleA_2.fastq.gz 12 ~/km_ALL_targets/ALL_targets/Fusion
bash run_km.sh ~/test_data/testSampleA_1.fastq.gz ~/test_data/testSampleA_2.fastq.gz 12 ~/km_ALL_targets/ALL_targets/focal_deletions
bash run_km.sh ~/test_data/testSampleA_1.fastq.gz ~/test_data/testSampleA_2.fastq.gz 12 ~/km_ALL_targets/ALL_targets/DUX4
bash run_km.sh ~/test_data/testSampleA_1.fastq.gz ~/test_data/testSampleA_2.fastq.gz 12 ~/km_ALL_targets/ALL_targets/IGH_fusion  # Note much longer run time.

ls output/testSampleA/
countTable31.jf  testSampleA_DUX4.txt  testSampleA_final_variants.csv  testSampleA_focal_deletions.txt  testSampleA_Fusion.txt  testSampleA_IGH_fusion.txt  testSampleA_SNV.txt
```

Producing the k-mer count table (countTable31.jf) is generally the rate limiting step of the process. The `run_km.sh` script therefore checks for the presence of the `countTable31.jf` first and if present does not re-generate this file but simply performs a `wc -l` command to read the data into memory.

The final target set 'IGH_fusion' contains the greatest number of targets given the highly variable nature of _IGH-CRLF2_ and _IGH-EPOR_ rearrangements. Run times against this target set are therefore substantially longer. For samples with 75b length and 60M read depth run-time ~6 mins for IGH_fusion targets vs a few seconds for testing other target sets, but can be much larger when using sequencing data of greater length/depth.

#### Collate results

The text file produced by km indicates results for all target sequences provided for a given mutation type. Many of these will be null results (no mutation identified). We have therefore written an R script that reads in these raw output files, filteres for identified fusions/sequence variants and provides some additional annotation (e.g. AA.Change for a detected SNV/InDel). The script will then write to the sample output directory a csv file containing idenified mutations of any type tested.

The R script requires the following libraries (tidyverse, Biostrings, rtracklayer). If not already present the script will install these required packages the frist time it is run.

4. To run on a single sample the script requires an argument specifying the relative path to the output directory for the sample under analysis.

Usage: Rscript bin/filter_km_output.R `[ouput/sampleDir]`

```
Rscript bin/filter_km_output.R output/testSampleA/
```

If no mutations of any type were identified then a csv file containing only the header row is produced.


## Run on multiple samples

ALL patients may posess a combination of genetic alterations of multiple types (SNVs, fusions, focal deletions) all contributing to disease. The overall aim of this package was to assess patients simultaneously for multiple mutation types, but limit the search for those variants which have been recurrently observed and are known to be clinically relevant. We therefore generally wish to assess multiple samples from a single sequencing run against all target sets. 

A helper script has been written for this purpose. This script requires a text file with locations of R1 and R2 for all samples to be tested. This script will perform run_km.sh on all samples for all target types except IGH_fusion and also collate results with the previously described R script. The script is set up to run targets within the km_ALL_targets/ALL_targets directory. If utilising targets in an alternate location the script will need to be modified accordingly.

1. To view the required arguments for helper_script.sh:

```
bash helper_script.sh

Usage: helper_script.sh [SAMPLE_SHEET] [THREADS]
Incorrect number of arguments
- Required: SAMPLE_SHEET = Tab-delimited Text file containing FASTQ R1 and R2 locations

<READ1 PATH> <--TAB--> <READ2 PATH>

- Optional: THREADS = Number of threads for running jellyfish. Default is 12.

```
2. To run `helper_script.sh` generate a sample_sheet containing the full/path/to/file of fastq files to be analysed and supply the location/fileName in the command:

```
cat sampleSheet.tsv 
/home/SAHMRI.INTERNAL/jacqueline.rehn/test_data/testSampleA_1.fastq.gz	/home/SAHMRI.INTERNAL/jacqueline.rehn/test_data/testSampleA_2.fastq.gz
/home/SAHMRI.INTERNAL/jacqueline.rehn/test_data/testSampleB_1.fastq.gz	/home/SAHMRI.INTERNAL/jacqueline.rehn/test_data/testSampleB_2.fastq.gz

bash helper_script.sh sampleSheet.tsv 10
```

## Generating custom fusion and variant targets

While the targets provided represent commonly observed fusions and clinically relevant sequence variants this list is not complete. ALL is a highly heterogenous disease and a large number of rare or novel fusions have been identified and implicated in disease development and progression. Furthermore, research is regularly identifying additional clinically relevant variants. To enable users to generate their own list of desired fusion/SNV/InDels for investigation we have generated two scripts for producing user defined fusions and sequence variants. This script requires a gtf file and associated reference genome.

To generate custom fusion targets a tab separated text file listing the genes, transcript identifier and exons involved in the fusion as well as desired length of targets is needed. Columns within the text file should represent: 
      1. Gene name of 5' fusion partner; 
      2. Transcript identifier for the 5' fusion partern transcript to be utilised for generating the target; 
      3. Exon range listing exons that are  likely break-points for 5' fusion partner; 
      4. Gene name of 3' fusion partner; 
      5. Transcript identifier for the 5' fusion partern transcript to be utilised for generating the target; 
      6. Exon range listing exons that are likely break-points for 3' fusion partner; 
      7. Desired target length (Note 62 is the minimum target length for k-mer size 31. Longer targets can be specified (e.g. 100 or 150 but must be an even number).

For example:

```
cat fusion_file.txt
gene1_symbol      gene1_id    gene1_exons gene2_symbol      gene2_id    gene2_exons target_length
EBF1  NM_024007   11:15 PDGFRB      NM_002609   9:11  62

```

The above text file would be used to generate 15 fusion target sequences for EBF1::PDGFRB, combining sequence from the EBF1 ref-seq transcript NM_024007 exons 11 to 15 with sequence from PDGFRB ref-seq transcript NM_002609 exons 9 to 11.

Note that the gene_id (transcript identifier) needs to be present in the gtf file and version you provide for creating target sequences. If utilising an ensembl gtf file then replace with the ensembl transcript_id for the gene of interest.

1. To view the required arguments for custom_fusion_targets.sh:

```
bash custom_fusion_targets.sh

Usage: custom_fusion_targets.sh [FUSION_FILE] [GTF] [REF] [OUT_DIR]

- Required: FUSION_FILE = Tab separated text file containing gene names, exons and target length

<GENE1_NAME> <--TAB--> <GENE1_EXONS> <--TAB--> <GENE2_NAME> <--TAB--> <GENE2_EXONS> <--TAB--> <TARGET_LENGTH>

- Required: GTF = Location of GTF annotation file
- Required: REF = Location of Reference genome
- Required: OUT_DIR = Location for writing custom targets.

```

This script currently accepts __UCSC__ and __Ensembl__ GTF and reference genomes. The user can either add the additional fusion targets to the ALL_targets/Fusion directory or specify an alternate directory location for writing generated target sequences.

2. To run custom_fusion_targets.sh using the above fusion_file.txt and add them the the ALL_targets/Fusion directory:

```
bash custom_fusion_targets.sh fusion_file.txt /path/to/hg19.refGene.gtf.gz /path/to/hg19.fa.gz ALL_targets/Fusion
```

To generate custom sequence variant targets a tab separated text file listing the gene name, transcript identifier and amino acid positions to target is required. Targets can be generated to span more than one amino acid position, but it is advised that the number of amino acids assessed in a give target be limited to 3-5. Targets _cannot_ be generated for the first 11 amino acids in a gene as this requires inclusion of bases from the 5' UTR.

Example text file containing information for generating custom variant targets:

```
cat variant_file.txt
gene_symbol transcript_id     pos1  pos2
PAX5  NM_016734   80    80
NRAS  NM_002524   12    13
```
As shown above, where only a single aminao acid position is desired from the target, this position should be repeated in column 4. Where the target sequence should span more than one amino acid position the first amino acid position should be placed in column 3 and the final amino acid position in column 4.

3. To view the required arguments for custom_fusion_targets.sh:

```
bash custom_variant_targets.sh

Usage: custom_variant_targets.sh [SNV_FILE] [GTF] [REF] [OUT_DIR]

- Required: SNV_FILE = Tab separated text file containing gene names and amino acid positions for target

<GENE1_NAME> <--TAB--> <AA_POS_1> <--TAB--> <GAA_POS_2>

- Required: GTF = Location of GTF annotation file
- Required: REF = Location of Reference genome
- Required: OUT_DIR = Location for writing custom targets.

```


4. To run custom_variant_targets.sh using the above SNVtargets.txt file and add them the the ALL_targets/SNV directory:

```
bash custom_variant_targets.sh SNVtargets.txt /path/to/hg19.refGene.gtf.gz /path/to/hg19.fa.gz ALL_targets/SNV
```


## Notes

- There will be `PathQuant.py:125: RuntimeWarning: invalid value encountered in greater_equal` warnings on km but these can be ignored
- Summary Rscript requires `tidyverse`, `Biostrings` and `here` r packages and are installed at the start of the script. If there are any issues, it might be worth running on your local copy of R (rather than conda env).

## Interpreting output

The `final_variants.csv` file has the following headers. 

```
cat output/testSampleA/testSampleA_final_variants.csv
"Target_Type","Alteration","Query","Type","Variant_name","rVAF","Expression","Min_coverage","Sequence","Reference_sequence"
```
Most of these fields are taken directly from the raw km output file while others are generated by the `filter_km_output.R` script. These fields indicate:

- Target_Type : Whether the mutation is a gene fusion (Fusion); single/di-nuclotide variant or indel (SNV); focal deletion (FocDel); DUX4 expression (DUX4); or IGH rearrangement (IGH) 
- Alteration : The annotated alteration.
      For a fusion the format is : gene1-gene2 identifed at the fusion break-point  e.g. BCR-ABL1 
      For an SNV/small variant  : gene AA.Change  e.g. CRLF2 F232C 
      For a focal deletion : gene exons-deleted e.g. IKZF1 4-7 del 
      For DUX4 expression : DUX4-rearrangement  (additional information given in other fields i.e. number of DUX4 targets (out of 9) detected & mean Min_coverage 
      For IGH : gene1-gene2 indentifed at/near potential breakpoint e.g. EPOR-IGHV4-39  
- Query : specific fasta file containing the target sequence for testing 
- Type : Type of mutation reported by km. If the target sequence is recreated (as in the case of fusions/DUX4) this will be 'Reference'. Otherwise can be 'Substitution', 'insertion', 'deletion', 'indel' or 'ITD' 
- Variant_name : Reported by km. Indicates sequence variation from the target sequence provided e.g. 35:t/G:36 to indicate t at base 35 of target sequence is substituted for G
- rVAF : Reported by km. Indicates ratio of coverage over reference path compared with alternate path representing a variant. For instances where expression of the reference target sequence is indicative of a mutation (e.g. fusions, DUX4 expression) this will be NA.
- Expression : Reported by km. 
- Min_coverage : Reported by km. Indicates the minimum number of k-mers covering any given position of the target sequence.
- Sequence : Reported by km. Is the sequence identified by overlapping k-mers beginning with the first 31b k-mer and ending with the final 31b k-mer of the target sequence.
- Reference_sequence : Reported by km. Is the sequence present in the target being queried.

