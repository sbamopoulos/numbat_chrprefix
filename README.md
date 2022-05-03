# Numbat

<!-- badges: start -->

[![<kharchenkolab>](https://circleci.com/gh/kharchenkolab/numbat.svg?style=svg)](https://app.circleci.com/pipelines/github/kharchenkolab/numbat)
  
<!-- badges: end -->

<img src="logo.png" align="right" width="150">

Numbat is a haplotype-enhanced CNV caller from single-cell transcriptomics data. It integrates signals from gene expression, allelic ratio, and population-derived haplotype information to accurately infer allele-specific CNVs in single cells and reconstruct their lineage relationship. 

Numbat can be used to:
 1. Detect allele-specific copy number variations from scRNA-seq 
 2. Differentiate tumor versus normal cells in the tumor microenvironment 
 3. Infer the clonal architecture and evolutionary history of profiled tumors. 

![image](https://user-images.githubusercontent.com/13375875/153020818-2e782689-09db-427f-ad98-2c175021a936.png)

Numbat does not require paired DNA or genotype data and operates solely on the donor scRNA-data data (for example, 10x Cell Ranger output). For details of the method, please checkout our preprint:

[Teng Gao, Ruslan Soldatov, Hirak Sarkar, et al. Haplotype-enhanced inference of somatic copy number profiles from single-cell transcriptomes. bioRxiv 2022.](https://www.biorxiv.org/content/10.1101/2022.02.07.479314v1)

# Table of contents
  
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Preparing data](#preparing-data)
- [Running Numbat](#running-numbat)
- [Understanding results](#understanding-results)

# Prerequisites
Numbat uses cellsnp-lite for generating SNP pileup data and eagle2 for phasing. Please follow their installation instructions and make sure their binary executables can be found in your $PATH.

1. [cellsnp-lite](https://github.com/single-cell-genetics/cellsnp-lite)
2. [eagle2](https://alkesgroup.broadinstitute.org/Eagle/)
3. [samtools](http://www.htslib.org/)

Additionally, Numbat needs a common SNP VCF and phasing reference panel. You can use the 1000 Genome reference below:

4. 1000G SNP VCF
```
# hg38
wget https://sourceforge.net/projects/cellsnp/files/SNPlist/genome1K.phase3.SNP_AF5e2.chr1toX.hg38.vcf.gz
# hg19
wget https://sourceforge.net/projects/cellsnp/files/SNPlist/genome1K.phase3.SNP_AF5e2.chr1toX.hg19.vcf.gz
```
4. 1000G Reference Panel
```
# hg38
wget http://pklab.med.harvard.edu/teng/data/1000G_hg38.zip
# hg19
wget http://pklab.med.harvard.edu/teng/data/1000G_hg19.zip
```
**Note**: currently Numbat only supports human hg19 and hg38 reference.

# Installation
You can install the numbat R package via:
```
devtools::install_github("https://github.com/kharchenkolab/numbat")
```
To get the most recent updates, you can install the development version:
```
devtools::install_github("https://github.com/kharchenkolab/Numbat/tree/devel")
```
# Preparing data
1. Prepare the allele data. Run the preprocessing script (`pileup_and_phase.R`) to count alleles and phase SNPs
```
usage: pileup_and_phase.R [-h] --label LABEL --samples SAMPLES --bams BAMS
                          --barcodes BARCODES --gmap GMAP [--eagle EAGLE]
                          --snpvcf SNPVCF --paneldir PANELDIR --outdir OUTDIR
                          --ncores NCORES [--UMItag UMITAG]
                          [--cellTAG CELLTAG] [--smartseq]

Run SNP pileup and phasing with 1000G

optional arguments:
  -h, --help           show this help message and exit
  --label LABEL        Individual label
  --samples SAMPLES    Sample names, comma delimited
  --bams BAMS          BAM files, one per sample, comma delimited
  --barcodes BARCODES  Cell barcode files, one per sample, comma delimited
  --gmap GMAP          Path to genetic map provided by Eagle2 (e.g. Eagle_v2.4.1/tables/genetic_map_hg38_withX.txt.gz)
  --snpvcf SNPVCF      SNP VCF for pileup
  --paneldir PANELDIR  Directory to phasing reference panel (BCF files)
  --outdir OUTDIR      Output directory
  --ncores NCORES      Number of cores
  --UMItag UMITAG      UMI tag in bam. Should be Auto for 10x and XM for
                       Slide-seq
  --cellTAG CELLTAG    Cell tag in bam. Should be CB for 10x and XC for Slide-
                       seq
  --smartseq           running with smart-seq mode
```
Note: If running with `--smartseq` mode, you may provide a file containing a list of bams (each as its own line) to `--bams` and a file containing a list of cell names (each as its own line) to `--barcodes`.

This will produce a file named `{sample}_allele_counts.tsv.gz` under the specified output directory, which contains cell-level phased allele counts. If multiple samples from the same individual was provided, there will be one allele count file for each sample. Other outputs include phased vcfs under `phasing/` folder and raw pileup counts under `pileup/`.

2. Prepare the expression data. Numbat takes a gene by cell integer UMI count matrix as input. You can directly use results from upstream transcriptome quantification pipelines such as 10x CellRanger.
  
3. Prepare the expression reference, which is a gene by cell type matrix of normalized expression values (raw gene counts divided by total counts). For a quick start, you may use a our HCA collection (`ref_hca`) that ships with the package. If you have matched normal cells (ideally, of various cell type) from the same patient or dataset and would like to make your own references, you may use this utility function:
```
# count_mat is a gene x cell raw count matrices
# cell_annot is a dataframe with columns "cell" and "cell_type"
ref_internal = aggregate_counts(count_mat, cell_annot)$exp_mat
```
  
# Running Numbat
  
In this example (ATC2 from [Gao et al](https://www.nature.com/articles/s41587-020-00795-2)), the gene expression count matrix and allele dataframe are already prepared for you.
```
library(numbat)

# run
out = run_numbat(
    count_mat = count_mat_ATC2, # gene x cell integer UMI count matrix 
    lambdas_ref = ref_hca, # reference expression profile, a gene x cell type normalized expression level matrix
    df_allele = df_allele_ATC2, # allele dataframe generated by pileup_and_phase script
    gtf = gtf_hg38, # provided upon loading the package
    genetic_map = genetic_map_hg38, # provided upon loading the package
    min_cells = 20,
    t = 1e-3,
    max_iter = 2,
    min_LLR = 50,
    init_k = 3,
    ncores = 10,
    plot = TRUE,
    out_dir = './test'
)
```
**Note**: If you wish to use your own custom reference, please use the `aggregate_counts` function as per the example in [preparing data](#preparing-data). You do not need to include the reference cells in `count_mat` or `df_allele`; only provide them as `lambdas_ref`.

## Run parameters
There are a few parameters you can consider tuning to a specific dataset. 
  
*CNV detection*
- `t`: the transition probability used in the HMM. A lower `t` is more appropriate for tumors with more complex copy number landscapes (from which you can expect more breakpoints) and is sometimes better for detecting subclonal events. A higher `t` is more effective for controlling false-positive rates of CNV calls.
- `gamma`: Overdispersion in allele counts (default: 20). For 10x data, 20 is recommended. Non-UMI protocols (e.g. SMART-Seq) usually produce noisier allele data, and a smaller value of gamma is recommended (e.g. 5). 
- `min_cells`: minimum number of cells for which an pseudobulk HMM will be run. If the allele coverage per cell is very sparse for a dataset, then I would consider setting this threshold to be higher.
- `multi_allelic`: Whether to enable calling of multiallelic CNVs
  
*CNV filtering*
- `min_LLR`: minimum log-likelihood ratio threshold to filter CNVs by. To ensure quality of phylogeny inference, we only use confident CNVs to reconstruct the phylogeny. By default, this threshold is 50.
- `max_entropy`: another criteria that we use to filter CNVs before phylogeny construction. The entropy of the binary posterior quantifies the uncertainty of an event across single cells. The value can be from 0 to 1 where 1 is the least stringent.
  
*Phylogeny*
- `tau`: Stringency to simplify the mutational history (0-1). A higher `tau` produces less clones and a more simplied evolutionary history.

*Iterative optimization*
- `init_k`: initial number of subclusters to use for the `hclust` initialization. Numbat by default uses hierarchical clustering (`hclust`) of smoothed expression values to approximate an initial phylogeny. This will cut the initial tree into k clusters. More clusters means more resolution at the initial stage for subclonal CNV detection. By default, we set init_k to be 3.
- `max_iter`: maximum number of iterations. Numbat iteratively optimizes the phylogeny and copy number estimations. In practice, we find that results after 2 iterations are usually stable.  
- `check_convergence`: stop iterating if the results have converged (based on consensus CNV segments).

*Parallelization*
- `ncores`: number of cores to use for single-cell CNV testing
- `ncores_nni`: number of cores to use for phylogeny inference
  
# Understanding results
A detailed vignette on how to interpret and visualize Numbat results is available:  
- [Interpreting Numbat results](https://kharchenkolab.github.io/numbat)
  
Numbat generates a number of files in the output folder. The file names are post-fixed with the `i`th iteration of phylogeny optimization. Here is a detailed list:
- `gexp_roll_wide.tsv.gz`: window-smoothed normalized expression profiles of single cells
- `hc.rds`: hierarchical clustering result based on smoothed expression
- `bulk_subtrees_{i}.tsv.gz`: pseudobulk HMM profiles based on subtrees defined by current cell lineage tree
- `segs_consensus_{i}.tsv.gz`: consensus segments from subtree pseudobulk HMMs
- `bulk_clones_{i}.tsv.gz`: pseudobulk HMM profiles based on clones defined by current cell lineage tree
- `bulk_clones_{i}.png`: visualization of clone pseudobulk HMM profiles
- `exp_sc_{i}.tsv.gz`: single-cell expression profiles used for single-cell CNV testing
- `exp_post_{i}.tsv`: single-cell expression posteriors 
- `allele_post_{i}.tsv`: single-cell allele posteriors 
- `joint_post_{i}.tsv`: single-cell joint posteriors 
- `treeUPGMA_{i}.rds`: UPGMA tree
- `treeNJ_{i}.rds`: NJ tree
- `tree_list_{i}.rds`: list of candidate phylogeneies in the maximum likelihood tree search
- `tree_final_{i}.rds`: final tree after simplification
- `mut_graph_{i}.rds`: final mutation history
- `clone_post_{i}.rds`: clone assignment and tumor versus normal classification posteriors
- `bulk_subtrees_{i}.png`: visualization of subtree pseudobulk HMM profiles 
- `bulk_clones_{i}.png`: visualization of clone pseudobulk HMM profiles 
- `panel_{i}.png`: visualization of combined phylogeny and CNV heatmap
