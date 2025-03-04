---
layout: tutorial_hands_on

title: ATAC-Seq data analysis
zenodo_link: https://zenodo.org/record/3862793
questions:
- Which DNA regions are accessible in the human lymphoblastoid cell line GM12878?
- How to analyse and visualise ATAC-Seq data?
objectives:
- Apply appropriate analysis and quality control steps for ATAC-Seq
- Generate a heatmap of transcription start site accessibility
- Visualise peaks for specific regions
time_estimation: 3h
key_points:
- ATAC-Seq can be used to identify accessible gene promoters and enhancers
- Several filters are applied to the reads, such as removing those mapped to mitochondria
- Fragment distribution can help determine whether an ATAC-Seq experiment has worked well
contributors:
- lldelisle
- mblue9
- heylf

---

# Introduction
{:.no_toc}

In many eukaryotic organisms, such as humans, the genome is tightly packed and organized with the help of nucleosomes (chromatin). A nucleosome is a complex formed by eight histone proteins that is wrapped with ~147bp of DNA. When the DNA is being actively transcribed into RNA, the DNA will be opened and loosened from the nucleosome complex. Many factors, such as the chromatin structure, the position of the nucleosomes, and histone modifications, play an important role in the organization and accessibility of the DNA. Consequently, these factors are also important for the activation and inactivation of genes. **A**ssay for **T**ransposase-**A**ccessible **C**hromatin using **seq**uencing ([ATAC-Seq](https://en.wikipedia.org/wiki/ATAC-seq)) is a method to investigate the accessibility of chromatin and thus a method to determine regulatory mechanisms of gene expression. The method can help identify promoter regions and potential enhancers and silencers. A promoter is the DNA region close to the transcription start site (TSS). It contains binding sites for transcription factors that will recruit the RNA polymerase. An enhancer is a DNA region that can be located up to 1 Mb downstream or upstream of the promoter. When transcription factors bind an enhancer and contact a promoter region, the transcription of the gene is increased. In contrast, a silencer decreases or inhibits the gene's expression. ATAC-Seq has become popular for identifying accessible regions of the genome as it's easier, faster and requires less cells than alternative techniques, such as FAIRE-Seq and DNase-Seq.

![ATAC-Seq](../../images/atac-seq/atac-seq.jpeg "Buenrostro et al. 2013 Nat Methods")

With ATAC-Seq, to find accessible (open) chromatin regions, the genome is treated with a hyperactive derivative of the Tn5 transposase. A [transposase](https://en.wikipedia.org/wiki/Transposase) can bind to a [transposable element](https://en.wikipedia.org/wiki/Transposable_element), which is a DNA sequence that can change its position (jump) within a genome (read the two links to get a deeper insight). During ATAC-Seq, the modified Tn5 inserts DNA sequences corresponding to truncated Nextera adapters into open regions of the genome and concurrently, the DNA is sheared by the transposase activity. The read library is then prepared for sequencing, including PCR amplification with full Nextera adapters and purification steps. Paired-end reads are recommended for ATAC-Seq for the reasons described [here](https://informatics.fas.harvard.edu/atac-seq-guidelines.html).

In this tutorial we will use data from the study of {% cite Buenrostro2013 %}, the first paper on the ATAC-Seq method. The data is from a human cell line of purified CD4+ T cells, called GM12878. The original dataset had 2 x 200 million reads and would be too big to process in a training session, so we downsampled the original dataset to 200,000 randomly selected reads. We also added about 200,000 reads pairs that will map to chromosome 22 to have a good profile on this chromosome, similar to what you might get with a typical ATAC-Seq sample (2 x 20 million reads in original FASTQ). Furthermore, we want to compare the predicted open chromatin regions to the known binding sites of CTCF, a DNA-binding protein implicated in 3D structure: [CTCF](https://en.wikipedia.org/wiki/CTCF). CTCF is known to bind to thousands of sites in the genome and thus it can be used as a positive control for assessing if the ATAC-Seq experiment is good quality. Good ATAC-Seq data would have accessible regions both within and outside of TSS, for example, at some CTCF binding sites. For that reason, we will download binding sites of CTCF identified by ChIP in the same cell line from ENCODE (ENCSR000AKB, dataset ENCFF933NTR).

### When working with real data
{:.no_toc}
When you use your own data we suggest you to use [this workflow](https://usegalaxy.eu/u/ldelisle/w/atac-seq-gtm-with-control) which includes the same steps but is compatible with replicates. If you do not have any control data you can import and edit this workflow, removing all steps with the controls. Controls for the ATAC-Seq procedure are not commonly performed, as discussed [here](https://informatics.fas.harvard.edu/atac-seq-guidelines.html), but could be ATAC-Seq of purified DNA.

> ### Agenda
>
> In this tutorial, we will cover:
>
> 1. TOC
> {:toc}
>
{: .agenda}


{% snippet faqs/galaxy/analysis_results_may_vary.md %}

# Preprocessing

## Get Data

We first need to download the sequenced reads (FASTQs) as well as other annotation files. Then, to increase the number of reads that will map to the reference genome (here human genome version 38, GRCh38/hg38), we need to preprocess the reads.


> ### {% icon hands_on %} Hands-on: Data upload
>
> 1. Create a new history for this tutorial
>
>    {% snippet faqs/galaxy/histories_create_new.md %}
>
> 2. Import the files from [Zenodo](https://doi.org/10.5281/zenodo.3862792) or from the shared data library
>
>    ```
>    https://zenodo.org/record/3862793/files/ENCFF933NTR.bed.gz
>    https://zenodo.org/record/3862793/files/SRR891268_chr22_enriched_R1.fastq.gz
>    https://zenodo.org/record/3862793/files/SRR891268_chr22_enriched_R2.fastq.gz
>    ```
>
>    {% snippet faqs/galaxy/datasets_import_via_link.md %}
>    {% snippet faqs/galaxy/datasets_import_from_data_library.md %}
>
> 3. Add a tag called `#SRR891268_R1` to the R1 file and a tag called `#SRR891268_R2` to the R2 file.
>
>    {% snippet faqs/galaxy/datasets_add_tag.md %}
>
> 4. Check that the datatype of the 2 FASTQ files is `fastqsanger.gz` and the peak file (ENCFF933NTR.bed.gz) is `encodepeak`. If they are not then change the datatype as described below.
>
>    {% snippet faqs/galaxy/datasets_change_datatype.md datatype="datatypes" %}
>
{: .hands_on}

> ### {% icon comment %} FASTQ format
> If you are not familiar with FASTQ format, see the [Quality Control tutorial]({% link topics/sequence-analysis/tutorials/quality-control/tutorial.md %})
{: .comment}
>
> ### {% icon comment %} BED / encode narrowPeak format
> If you are not familiar with BED format or encode narrowPeak format, see the [BED Format](https://genome.ucsc.edu/FAQ/FAQformat.html)
{: .comment}

We will visualise regions later in the analysis and obtain the gene information now. We will get information for chromosome 22 genes (names of transcripts and genomic positions) using the UCSC tool.

> ### {% icon hands_on %} Hands-on: Obtain Annotation for hg38 genes
>
> 1. **UCSC Main** {% icon tool %} with the following parameters:
>    - *"clade"*: `Mammal`
>    - *"genome"*: `Human`
>    - *"assembly"*: `Dec. 2013 (GRCh38/hg38)`
>    - *"group"*: `Genes and Gene Prediction`
>    - *"track"*: `All GENCODE V37`
>    - *"table"*: `Basic`
>    - *"region"*: `position` `chr22`
>    - *"output format"*: `all fields from selected table`
>    - *"Send output to"*: `Galaxy`
> 2. Click **get output**
> 3. Click **Send query to Galaxy**
>
>    This table contains all the information but is not in a BED format. To transform it into BED format we will cut out the required columns and rearrange:
>
> 4. **Cut** columns from a table {% icon tool %} with the following parameters:
>    - {% icon param-text %} *"Cut columns"*: `c3,c5,c6,c13,c12,c4`
>    - {% icon param-text %} *"Delimited by"*: `Tab`
>    - {% icon param-file %} *"From"*: `UCSC Main on Human: wgEncodeGencodeBasicV37 (chr22:1-50,818,468)`
>
> 5. Check the contents of your file, is this as you expect it to be?
>
>    > ### {% icon question %} Question: Expected output
>    >
>    > Our goal here was to convert the data to BED format.
>    >
>    > 1. Which columns do you expect in your file? (Tip: read about [BED format](https://genome.ucsc.edu/FAQ/FAQformat.html#format1))
>    > 2. Does your file look like a valid BED format?
>    >
>    > > ### {% icon solution %} Solution
>    > >
>    > > 1. We expect at least 3 columns, `chromosome - start - end`, and possibly more optional columns
>    > > 2. Your file should look something like this:
>    > >    ```
>    > >    chr22	10736170	10736283	U2	0	-
>    > >    chr22	11066417	11068174	CU104787.1	0	+
>    > >    chr22	11249808	11249959	5_8S_rRNA	0	-
>    > >    chr22	15273854	15273961	U6	0	+
>    > >    [..]
>    > >    ```
>    > >
>    > > - **Troubleshooting:** Is your second column the `Strand` column?
>    > >    - Make sure you used the correct **Cut** {% icon tool %} (the one that matches the tool name mentioned in the previous step *exactly*)
>    > >    - There is another tool with `(cut)` behind the title, we do NOT want to use this tool in this step.
>    > >
>    > > - **Tip:** Always check your output files to make sure they match your expectations!
>    > >
>    > {: .solution}
>    >
>    {: .question}
>
>
> 6. **Rename** {% icon galaxy-pencil %} the dataset as `chr22 genes`
>
>    {% snippet faqs/galaxy/datasets_rename.md %}
>
> 7. **Change** {% icon galaxy-pencil %} its datatype to BED
>
>    {% snippet faqs/galaxy/datasets_change_datatype.md datatype="bed" %}
>
> 8. Click on the {% icon galaxy-eye %} (eye) icon of the file. It should have now column names and they should match the content.
>
{: .hands_on}

>
> ### {% icon comment %} Gene file
> The chr22 genes BED we produced only contains the start, the end, the name, and the strand of each transcript. It does not contain exon information.
> To be able to have the exon information, you could use a GTF file which can be downloaded from the [gencode website](ftp://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_37/gencode.v37.annotation.gtf.gz) but this file would include the information for the whole genome and would slow the analysis.
{: .comment}

## Quality Control

The first step is to check the quality of the reads and the presence of the Nextera adapters. When we perform ATAC-Seq, we can get DNA fragments of about 40 bp if two adjacent Tn5 transposases cut the DNA {% cite Adey2010 %}. This can be smaller than the sequencing length so we expect to have Nextera adapters at the end of those reads. We can assess the reads with **FastQC**.

> ### {% icon hands_on %} Hands-on: Task description
>
> 1. **FastQC** {% icon tool %} with the default parameters:
>       - *"Short read data from your current history"*: Choose here either only the `SRR891268_R1` file with {% icon param-file %} or use {% icon param-files %} **Multiple datasets** to choose both `SRR891268_R1` and `SRR891268_R2`.
> 2. Inspect the web page output of **FastQC** {% icon tool %} for the `SRR891268_R1` sample. Check what adapters are found at the end of the reads.
>
> > ### {% icon question %} Questions
> >
> > 1. How many reads are in the FASTQ?
> > 2. Which sections have a warning?
> >
> > > ### {% icon solution %} Solution
> > >
> > > 1. There are 285247 reads.
> > > 2. The 3 steps below have warnings:
> > >
> > >    1. **Per base sequence content**
> > >
> > >       It is well known that the Tn5 has a strong sequence bias at the insertion site. You can read more about it in {% cite Green2012 %}.
> > >
> > >    2. **Sequence Duplication Levels**
> > >
> > >       The read library quite often has PCR duplicates that are introduced
> > >       simply by the PCR itself. We will remove these duplicates later on.
> > >
> > >    3. **Overrepresented sequences**
> > >
> > >       One sequence is over represented: 
> > >       you have 306 reads which are exactly the sequence of the Nextera adapter.
> > >       They correspond to adapters amplified head-to-head.
> > >       306 is really low (only 0.1% of reads).
> > >
> > {: .solution}
> >
>    {: .question}
{: .hands_on}

> ### {% icon comment %} FastQC Results
> This is what you should expect from the **Adapter Content** section:
> ![FastQC screenshot of the Adapter Content section](../../images/atac-seq/Screenshot_fastqcBeforecutadapt.png "FastQC screenshot on the Adapter Content section")
{: .comment}

The FastQC web page **Adapter Content** section shows the presence of Nextera Transposase Sequence in the reads. We will remove the adapters with Cutadapt.

## Trimming Reads

To trim the adapters we provide the Nextera adapter sequences to **Cutadapt**. These adapters are shown in the image below.

![Nextera library with the sequence of adapters](../../images/atac-seq/nexteraLibraryPicture.svg.png "Nextera library with the sequence of adapters")

The forward and reverse adapters are slightly different. We will also trim low quality bases at the ends of the reads (quality less than 20). We will only keep reads that are at least 20 bases long. We remove short reads (< 20bp) as they are not useful, they will either be thrown out by the mapping or may interfere with our results at the end.


> ### {% icon hands_on %} Hands-on: Task description
>
> 1. **Cutadapt** {% icon tool %} with the following parameters:
>    - *"Single-end or Paired-end reads?"*: `Paired-end`
>        - {% icon param-file %} *"FASTQ/A file #1"*: select `SRR891268_R1`
>        - {% icon param-file %} *"FASTQ/A file #2"*: select `SRR891268_R2`
>        - In *"Read 1 Options"*:
>            - In *"3' (End) Adapters"*:
>                - {% icon param-repeat %} *"Insert 3' (End) Adapters"*
>                    - *"Source"*: `Enter custom sequence`
>                        - {% icon param-text %} *"Enter custom 3' adapter name (Optional if Multiple output is 'No')"*: `Nextera R1`
>                        - {% icon param-text %} *"Enter custom 3' adapter sequence"*: `CTGTCTCTTATACACATCTCCGAGCCCACGAGAC`
>        - In *"Read 2 Options"*:
>            - In *"3' (End) Adapters"*:
>                - {% icon param-repeat %} *"Insert 3' (End) Adapters"*
>                    - *"Source"*: `Enter custom sequence`
>                        - {% icon param-text %} *"Enter custom 3' adapter name (Optional)"*: `Nextera R2`
>                        - {% icon param-text %} *"Enter custom 3' adapter sequence"*: `CTGTCTCTTATACACATCTGACGCTGCCGACGA`
>    - In *"Filter Options"*:
>        - {% icon param-text %} *"Minimum length"*: `20`
>    - In *"Read Modification Options"*:
>        - {% icon param-text %} *"Quality cutoff"*: `20`
>    - In *"Output Options"*:
>        - *"Report"*: `Yes`
>
> 2. Click on the {% icon galaxy-eye %} (eye) icon of the report and read the first lines.
{: .hands_on}

> ### {% icon comment %} Cutadapt Results
> You should get similar output to this from Cutadapt:
> ![Summary of cutadapt](../../images/atac-seq/Screenshot_cutadaptSummary.png "Summary of cutadapt")
{: .comment}


> ### {% icon question %} Questions
>
> 1. What percentage of reads contain adapters?
> 2. What percentage of reads are still longer than 20bp after the trimming?
>
> > ### {% icon solution %} Solution
> >
> > 1. ~14%
> > 2. ~99%
> >
> {: .solution}
>
{: .question}

> ### {% icon hands_on %} Hands-on: Check Adapter Removal with FastQC
>
> 1. **FastQC** {% icon tool %} with the default parameters:
>       - *"Short read data from your current history"*: select the output of **Cutadapt** {% icon param-files %} **Multiple datasets** to choose both `Read 1 Output` and `Read 2 Output`.
>
> 2. Click on the {% icon galaxy-eye %} (eye) icon of the report and read the first lines.
{: .hands_on}

> ### {% icon comment %} FastQC Results
> Now, you should see under **Overrepresented sequences** that there is no more overrepresented sequences and under **Adapter Content** that the Nextera adapters are no longer present.
> ![FastQC screenshot on the adapter content section after cutadapt](../../images/atac-seq/Screenshot_fastqcAftercutadapt.png "FastQC screenshot on the adapter content section after cutadapt")
> However, you may have noticed that you have a new section with warning: **Sequence Length Distribution**. This is expected as you trimmed part of the reads.
{: .comment}

# Mapping

## Mapping Reads to Reference Genome

Next we map the trimmed reads to the human reference genome. Here we will use **Bowtie2**. We will extend the maximum fragment length (distance between read pairs) from 500 to 1000 because we know some valid read pairs are from this fragment length. We will use the `--very-sensitive` parameter to have more chance to get the best match even if it takes a bit longer to run. We will run the **end-to-end** mode because we trimmed the adapters so we expect the whole read to map, no clipping of ends is needed. Regarding the genome to choose. The hg38 version of the human genome contains [alternate loci](https://www.ncbi.nlm.nih.gov/grc/help/definitions/#ALTERNATE). This means that some region of the genome are present both in the canonical chromosome and on its alternate loci. The reads that map to these regions would map twice. To be able to filter reads falling into repetitive regions but keep reads falling into regions present in alternate loci, we will map on the Canonical version of hg38 (only the chromosome with numbers, chrX, chrY, and chrM).

> ### {% icon comment %} Dovetailing
> We will allow dovetailing of read pairs with Bowtie2. This is because adapters are removed by Cutadapt only when at least 3 bases match the adapter sequence, so it is possible that after trimming a read can contain 1-2 bases of adapter and go beyond it's mate start site. For example, if the first mate in the read pair is: `GCTATGAAGAATAGGGCGAAGGGGCCTGCGGCGTATTCGATGTTGAAGCT` and the second mate is `CTTCAACATCGAATACGCCGCAGGCCCCTTCGCCCTATTCTTCATAGCCT`, where both contain 2 bases of adapter sequence, they will not be trimmed by Cutadapt and will map this way:
> ```
<--------------------Mate 1-----------------------
AGCTTCAACATCGAATACGCCGCAGGCCCCTTCGCCCTATTCTTCATAGC
  CTTCAACATCGAATACGCCGCAGGCCCCTTCGCCCTATTCTTCATAGCCT
  ----------------------Mate 2--------------------->
```
This is what we call dovetailing and we want to consider this pair as a valid concordant alignment.
{: .comment}


> ### {% icon hands_on %} Hands-on: Mapping reads to reference genome
>
> 1. **Bowtie2** {% icon tool %} with the following parameters:
>    - *"Is this single or paired library"*: `Paired-end`
>        - {% icon param-file %} *"FASTQ/A file #1"*: select the output of **Cutadapt** {% icon tool %} *"Read 1 Output"*
>        - {% icon param-file %} *"FASTQ/A file #2"*: select the output of **Cutadapt** {% icon tool %} *"Read 2 Output"*
>        - *"Do you want to set paired-end options?"*: `Yes`
>            - *"Set the maximum fragment length for valid paired-end alignments"*: `1000`
>            - *"Allow mate dovetailing"*: `Yes`
>    - *"Will you select a reference genome from your history or use a built-in index?"*: `Use a built-in genome index`
>        - *"Select reference genome"*: `Human (Homo sapiens): hg38 Canonical`
>    - *"Select analysis mode"*: `1: Default setting only`
>        - *"Do you want to use presets?"*: `Very sensitive end-to-end (--very-sensitive)`
>    - *"Save the bowtie2 mapping statistics to the history"*: `Yes`
>
> 2. Click on the {% icon galaxy-eye %} (eye) icon of the mapping stats.
{: .hands_on}

> ### {% icon comment %} Bowtie2 Results
> You should get similar results to this from Bowtie2:
> ![Mapping statistics of bowtie2](../../images/atac-seq/Screenshot_bowtie2MappingStats.png "Mapping statistics of bowtie2")
{: .comment}

> ### {% icon question %} Questions
>
> What percentage of read pairs mapped concordantly?
>
> > ### {% icon solution %} Solution
> >
> > 54.8+42.87=97.67%
> >
> {: .solution}
>
{: .question}

> ### {% icon comment %} Comment on the number of uniquely mapped.
>
> You might be surprised by the number of uniquely mapped compared to the number of multi-mapped reads (reads mapping to more than one location in the genome).
> One of the reasons is that we have used the parameter `--very-sensitive`. Bowtie2 considers a read as multi-mapped even if the second hit has a much lower quality than the first one.
> Another reason is that we have reads that map to the mitochondrial genome. The mitochondrial genome has a lot of regions with similar sequence.
>
{: .comment}

# Filtering Mapped Reads

## Filter Uninformative Reads

We apply some filters to the reads after the mapping. ATAC-Seq datasets can have a lot of reads that map to the mitchondrial genome because it is nucleosome-free and thus very accessible to Tn5 insertion. The mitchondrial genome is uninteresting for ATAC-Seq so we remove these reads. We also remove reads with low mapping quality and reads that are not properly paired.


> ### {% icon hands_on %} Hands-on: Filtering of uninformative reads
>
> 1. **Filter** BAM datasets on a variety of attributes {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"BAM dataset(s) to filter"*: Select the output of  **Bowtie2** {% icon tool %} *"alignments"*
>    - In *"Condition"*:
>        - *"1: Condition"*
>            - In *"Filter"*:
>                - *"1: Filter"*
>                    - *"Select BAM property to filter on"*: `mapQuality`
>                        - *"Filter on read mapping quality (phred scale)"*: `>=30`
>                - {% icon param-repeat %} *"Insert Filter"*
>                    - *"Select BAM property to filter on"*: `isProperPair`
>                        - *"Select properly paired reads"*: `Yes`
>                - {% icon param-repeat %} *"Insert Filter"*
>                    - *"Select BAM property to filter on"*: `reference`
>                        - *"Filter on the reference name for the read"*: `!chrM`
>    - *"Would you like to set rules?"*: `No`
>
>
> 2. Click on the input and the output BAM files of the filtering step. Check the size of the files.
>
{: .hands_on}

> ### {% icon question %} Questions
>
> 1. Based on the file size, what proportion of alignments was removed (approximately)?
> 2. Which parameter should be modified if you are interested in repetitive regions?
>
> > ### {% icon solution %} Solution
> >
> > 1. The original BAM file is 28.1 MB, the filtered one is 15.2 MB. Approximately half of the alignments were removed.
> >
> > 2. You should modify the mapQuality criteria and decrease the threshold.
> >
> {: .solution}
>
{: .question}

High numbers of mitochondrial reads can be a problem in ATAC-Seq. Some ATAC-Seq samples have been reported to be 80% mitochondrial reads and so wet-lab methods have been developed to deal with this issue {% cite Corces2017 %} and {% cite Litzenburger2017 %}. It can be a useful QC to assess the number of mitochondrial reads.
However, it does not predict the quality of the rest of the data. It is just that sequencing reads have been wasted.

> ### {% icon tip %} Tip: Getting the number of mitochondrial reads
>
> To get the number of reads that mapped to the mitochondrial genome (chrM) you can run **Samtools idxstats** {% icon tool %} on the output of  **Bowtie2** {% icon tool %} *"alignments"*.
> The columns of the output are: chromosome name, chromosome length, number of reads mapping to the chromosome, number of unaligned mate whose mate is mapping to the chromosome.
> The first 2 lines of the result would be (after using **Sort** {% icon tool %}):
>
> ![Samtools idxstats result](../../images/atac-seq/Screenshot_samtoolsIdxStatsChrM.png "Samtools idxstats result")
>
> There are 220 000 reads which map to chrM and 165 000 which map to chr22.
{: .tip}

## Filter Duplicate Reads

Because of the PCR amplification, there might be read duplicates (different reads mapping to exactly the same genomic region) from overamplification of some regions. As the Tn5 insertion is random within an accessible region, we do not expect to see fragments with the same coordinates. We consider such fragments to be PCR duplicates. We will remove them with **Picard MarkDuplicates**.

> ### {% icon hands_on %} Hands-on: Remove duplicates
>
> 1. **MarkDuplicates** {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Select SAM/BAM dataset or dataset collection"*: Select the output of  **Filter** {% icon tool %} *"BAM"*
>    - *"If true do not write duplicates to the output file instead of writing them with appropriate flags set"*: `Yes`
>
>    > ### {% icon comment %} Comment: Default of  **MarkDuplicates** {% icon tool %}
>    >
>    > By default, the tool will only "Mark" the duplicates. This means that it will change the Flag of the duplicated reads to enable them to be filtered afterwards. We use the parameter *"If true do not write duplicates to the output file instead of writing them with appropriate flags set"* to directly remove the duplicates.
>    {: .comment}
>
> 2. Click on the {% icon galaxy-eye %} (eye) icon of the MarkDuplicate metrics.
{: .hands_on}

> ### {% icon comment %} MarkDuplicates Results
> You should get similar output to this from MarkDuplicates:
> ![Metrics of MarkDuplicates](../../images/atac-seq/Screenshot_picardRemoveDup.png "Metrics of MarkDuplicates")
{: .comment}

> ### {% icon tip %} Tip: Formatting the MarkDuplicate metrics for readability
>
> 1. **Select** lines that match an expression {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Select lines from"*: Select the output of  **MarkDuplicates** {% icon tool %}
>    - *"that*: `Matching`
>    - *"the pattern*: `(Library|LIBRARY)`
> 2. Check that the datatype is tabular. If not, change it.
>    {% snippet faqs/galaxy/datasets_change_datatype.md datatype="tabular" %}
> 3. **Transpose** {% icon tool %}:
>    - {% icon param-file %} *"Select lines from"*: Select the output of **Select** {% icon tool %}
>
> ![Metrics of MarkDuplicates](../../images/atac-seq/Screenshot_picardRemoveDupAfterTranspose.png "Metrics of MarkDuplicates")
>
{: .tip}

> ### {% icon question %} Questions
>
> 1. How many pairs were in the input?
> 2. How many pairs are duplicates?
>
> > ### {% icon solution %} Solution
> >
> > 1. 135813
> > 2. 3584
> >
> {: .solution}
>
{: .question}

Once again, if you have a high number of replicates it does not mean that your data are not good, it just means that you sequenced too much compared to the diversity of the library you generated. Consequently, libraries with a high portion of duplicates should not be resequenced as this would not increase the amount of data.

## Check Insert Sizes

We will check the insert sizes with **Paired-end histogram** of insert size frequency. The insert size is the distance between the R1 and R2 read pairs. This tells us the size of the DNA fragment the read pairs came from. The fragment length distribution of a sample gives a very good indication of the quality of the ATAC-Seq.

> ### {% icon hands_on %} Hands-on: Plot the distribution of fragment sizes.
>
> 1. **Paired-end histogram** of insert size frequency {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"BAM file"*: Select the output of  **MarkDuplicates** {% icon tool %} *"BAM output"*
>    - *"Lower bp limit (optional)"*: `0`
>    - *"Upper bp limit (optional)"*: `1000`
>
> 2. Click on the {% icon galaxy-eye %} (eye) icon of the lower one of the 2 outputs (the png file).
{: .hands_on}

> ### {% icon comment %} CollectInsertSizeMetrics Results
> This is what you get from CollectInsertSizeMetrics:
> ![Fragment size distribution](../../images/atac-seq/Screenshot_sizeDistribution.png "Fragment size distribution")
{: .comment}

> ### {% icon question %} Questions
>
> Could you guess what the peaks at approximately 50bp, 200bp, 400bp and 600bp correspond to?
>
> > ### {% icon solution %} Solution
> >
> > The first peak (50bp) corresponds to where the Tn5 transposase inserted into nucleosome-free regions. The second peak (a bit less than 200bp) corresponds to where Tn5 inserted around a single nucleosome. The third one (around 400bp) is where Tn5 inserted around two adjacent nucleosomes and the fourth one (around 600bp) is where Tn5 inserted around three adjacent nucleosomes.
> >
> {: .solution}
>
{: .question}

This fragment size distribution is a good indication if your experiment worked or not.
In absence of chromatin (without nucleosome), this is the profile you would get:

![Fragment size distribution of a purified DNA](../../images/atac-seq/Screenshot_sizeDistribution_Naked.png "Fragment size distribution of a purified DNA")

Here are examples of Fragment size distributions of ATAC-Seq which were very noisy:

![Fragment size distribution of a failed ATAC-Seq](../../images/atac-seq/Screenshot_sizeDistribution_Failed.png "Fragment size distribution of a failed ATAC-Seq")

![Fragment size distribution of another failed ATAC-Seq](../../images/atac-seq/Screenshot_sizeDistribution_Failed2.png "Fragment size distribution of another very noisy ATAC-Seq")

A final example of a Fragment size distribution of a very good ATAC-Seq, even if we cannot see the third nucleosome "peak".
![Fragment size distribution of a good ATAC-Seq](../../images/atac-seq/Screenshot_sizeDistribution_Good.png "Fragment size distribution of a good ATAC-Seq")


# Peak calling

## Call Peaks

We have now finished the data preprocessing. Next, in order to find regions corresponding to potential open chromatin regions, we want to identify regions where reads have piled up (peaks) greater than the background read coverage. The tools which are currently used are [Genrich](https://github.com/jsh58/Genrich) and [MACS2](https://github.com/taoliu/MACS). MACS2 is more widely used. Genrich has a mode dedicated to ATAC-Seq but is still not published and the more reads you have, the less peaks you get (see the issue [here](https://github.com/jsh58/Genrich/issues/33)). That's why we will not use Genrich in this tutorial.

At this step, two approaches exists:

- The first one is to select only paired whose fragment length is below 100bp corresponding to nucleosome-free regions and to use a peak calling like you would do for a ChIP-seq, joining signal between mates. The disadvantages of this approach is that you can only use it if you have paired-end data and you will miss small open regions where only one Tn5 bound.
- The second one chosen here is to use all reads to be more exhaustive. In this approach, it is very important to re-center the signal of each reads on the 5' extremity (read start site) as this is where Tn5 cuts. Indeed, you want your peaks around the nucleosomes and not directly on the nucleosome:
![Scheme of ATAC-Seq reads relative to nucleosomes](../../images/atac-seq/schemeWithLegend.jpg "Scheme of ATAC-Seq reads relative to nucleosomes")

> ### {% icon comment %} Comment on Tn5 insertion
>
> When Tn5 cuts an accessible chromatin locus it inserts adapters separated by 9bp ({% cite Kia2017 %}):
> ![Nextera Library Construction](../../images/atac-seq/NexteraLibraryConstruction.jpg "Nextera Library Construction")
>
> This means in order to have the read start site reflecting the centre of where Tn5 bound, the reads on the positive strand should be shifted 4 bp to the right and reads on the negative strands should be shifted 5 bp to the left as in [Buenrostro et al. 2013](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3959825). **Genrich** can apply these shifts when ATAC-seq mode is selected. In most cases, we do not have 9bp resolution so we don't take it into account but if you are interested in the footprint, this is important.
{: .comment}

If we only assess the coverage of the 5' extremity of the reads, the data would be too sparse and it would be impossible to call peaks. Thus, we will extend the start sites of the reads by 200bp (100bp in each direction) to assess coverage.

### Using MACS2

We convert the BAM file to BED format because when we set the extension size in MACS2, it will only consider one read of the pair while here we would like to use the information from both.

> ### {% icon hands_on %} Hands-on: Convert the BAM to BED
>
> 1. **bedtools BAM to BED** converter {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Convert the following BAM file to BED"*: Select the output of **MarkDuplicates** {% icon tool %}
>
{: .hands_on}

We call peaks with MACS2. In order to get the coverage centered on the 5' extended 100bp each side we will use `--shift -100` and `--extend 200`:
![MACS2 options to get 100bp each side](../../images/atac-seq/macs2Options.jpg "MACS2 options to get 100bp each side")


> ### {% icon hands_on %} Hands-on: Call peaks with MACS2
>
> 1. **MACS2 callpeak** {% icon tool %} with the following parameters:
>    - *"Are you pooling Treatment Files?"*: `No`
>    - *"ChIP-Seq Treatment File"*: Select the output of **bedtools BAM to BED** converter {% icon tool %}
>    - *"Do you have a Control File?"*: `No`
>    - *"Format of Input Files"*: `Single-end BED`
>    - *"Effective genome size"*: `H. sapiens (2.7e9)`
>    - *"Build Model"*: `Do not build the shifting model (--nomodel)`
>        - *"Set extension size"*: `200`
>        - *"Set shift size"*: `-100`. It needs to be - half the extension size to be centered on the 5'.
>    - *"Additional Outputs"*:
>        - Check `Peaks as tabular file (compatible with MultiQC)`
>        - Check `Peak summits`
>        - Check `Scores in bedGraph files`
>    - In *"Advanced Options"*:
>        - *"Composite broad regions"*: `No broad regions`
>            - *"Use a more sophisticated signal processing approach to find subpeak summits in each enriched peak region"*: `Yes`
>        - *"How many duplicate tags at the exact same location are allowed?"*: `all`
>
>    > ### {% icon comment %} Why keeping all duplicates is important
>    >
>    > We previously removed duplicates using **MarkDuplicates** {% icon tool %} using paired-end information. If two pairs had identical R1 but different R2, we knew it was not a PCR duplicate. Because we converted the BAM to BED we lost the pair information. If we keep the default (removing duplicates) one of the 2 identical R1 would be filtered out as duplicate.
>    {: .comment}
>
{: .hands_on}

# Visualisation of Coverage

## Prepare the Datasets

### Extract CTCF peaks on chr22 in intergenic regions
As our training dataset is focused on chromosome 22 we will only use the CTCF peaks from chr22. We expect to have ATAC-seq coverage at TSS but only good ATAC-seq have coverage on intergenic CTCF. Indeed, the CTCF protein is able to position nucleosomes and creates a region depleted of nucleosome of around 120bp {% cite fu_insulator_2008 %}. This is smaller than the 200bp nucleosome-free region around TSS and also probably not present in all cells. Thus it is more difficult to get enrichment. In order to get the list of intergenic CTCF peaks of chr22, we will first select the peaks on chr22 and then exclude the one which overlap with genes.

> ### {% icon hands_on %} Hands-on: Select CTCF peaks from chr22 in intergenic regions:
>
> 1. **Filter** data on any column using simple expressions {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Filter"*: Select the first dataset: `ENCFF933NTR.bed.gz`
>    - *"With following condition"*: `c1=='chr22'`
>
> 2. **bedtools Intersect intervals** find overlapping intervals in various ways {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"File A to intersect with B"*: Select the output of **Filter** data on any column using simple expressions {% icon tool %}
>    - *Combined or separate output files*:
>        - {% icon param-file %} *"File B to intersect with A"*: Select the dataset `chr22 genes`
>    - *What should be written to the output file?*: `Write the original entry in A for each overlap (-wa)`
>    - Report only those alignments that **do not** overlap with file(s) B: `Yes`
>
> 3. Rename the datasets `intergenic CTCF peaks chr22`.
{: .hands_on}


### Convert bedgraph from **MACS2** to bigwig
The bedgraph format is easily readable for human but it can be very large and visualising a specific region is quite slow. We will change it to bigwig format which is a binary format, so we can visualise any region of the genome very quickly.

> ### {% icon hands_on %} Hands-on: Convert bedgraphs to bigWig.
>
> 1. **Wig/BedGraph-to-bigWig** converter {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Convert"*: Select the output of **MACS2** {% icon tool %} (Bedgraph Treatment).
>    - *"Converter settings to use"*: `Default`
>
> 2. Rename the datasets `MACS2 bigwig`.
{: .hands_on}


## Create heatmap of coverage at TSS with deepTools

You might be interested in checking the coverage on specific regions. For this, you can compute a heatmap. We will use the **deepTools plotHeatmap**. As an example, we will here make a heatmap centered on the transcription start sites (TSS) and another one centered on intergenic CTCF peaks. First, on the TSS:

### Generate computeMatrix

The input of **plotHeatmap** is a matrix in a hdf5 format. To generate it we use the tool **computeMatrix** that will evaluate the coverage at each locus we are interested in.

> ### {% icon hands_on %} Hands-on: Generate the matrix
>
> 1. **computeMatrix** {% icon tool %} with the following parameters:
>    - In *"Select regions"*:
>        - 1. *"Select regions"*
>            - {% icon param-file %} *"Regions to plot"*: Select the dataset `chr22 genes`
>    - *"Sample order matters"*: `No`
>        - {% icon param-file %} *"Score file"*: Select the output of **Wig/BedGraph-to-bigWig** {% icon tool %} that should be named `MACS2 bigwig`.
>    - *"computeMatrix has two main output options"*: `reference-point`
>    - *"The reference point for the plotting"*: `beginning of region (e.g. TSS)`
>    - *"Show advanced output settings"*: `no`
>    - *"Show advanced options"*: `yes`
>        - *"Convert missing values to 0?"*: `yes`
>
{: .hands_on}


### Plot with **plotHeatmap**

We will now generate a heatmap. Each line will be a transcript. The coverage will be summarized with a color code from red (no coverage) to blue (maximum coverage). All TSS will be aligned in the middle of the figure and only the 2 kb around the TSS will be displayed. Another plot, on top of the heatmap, will show the mean signal at the TSS. There will be one heatmap per bigwig.

> ### {% icon hands_on %} Hands-on: Generate the heatmap
>
> 1. **plotHeatmap** {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Matrix file from the computeMatrix tool"*: Select the output of **computeMatrix** {% icon tool %}.
>    - *"Show advanced output settings"*: `no`
>    - *"Show advanced options"*: `no`
{: .hands_on}

> ### {% icon comment %} plotHeatmap Results
> This is what you get from plotHeatmap:
> ![plotHeatmap output](../../images/atac-seq/plotHeatmapOutput.png "plotHeatmap output")
{: .comment}

> ### {% icon question %} Questions
>
> 1. Is the coverage symmetric?
> 2. What is the mean value in genes?
>
> > ### {% icon solution %} Solution
> >
> > 1. No, it is higher on the left which is expected as usually the promoter of active genes is accessible.
> > 2. Around 5.5.
> >
> {: .solution}
>
{: .question}

Now we will repeat the procedure for CTCF peaks of chr22 in intergenic regions:

> ### {% icon hands_on %} Hands-on: Generate the matrix
>
> 1. **computeMatrix** {% icon tool %} with the following parameters:
>    - In *"Select regions"*:
>        - 1. *"Select regions"*
>            - {% icon param-file %} *"Regions to plot"*: Select the dataset `intergenic CTCF peaks chr22`
>    - *"Sample order matters"*: `No`
>        - {% icon param-file %} *"Score file"*: Select the output of **Wig/BedGraph-to-bigWig** {% icon tool %} that should be named `MACS2 bigwig`.
>    - *"computeMatrix has two main output options"*: `reference-point`
>    - *"The reference point for the plotting"*: `center of region`
>    - *"Show advanced output settings"*: `no`
>    - *"Show advanced options"*: `yes`
>        - *"Convert missing values to 0?"*: `yes`
>
> 2. **plotHeatmap** {% icon tool %} with the following parameters:
>    - {% icon param-file %} *"Matrix file from the computeMatrix tool"*: Select the output of **computeMatrix** {% icon tool %}.
>    - *"Show advanced output settings"*: `no`
>    - *"Show advanced options"*: `yes`
>        - *"Colormap to use for each sample"*:
>            - {% icon param-repeat %} *"Insert Colormap to use for each sample"*:
>                - *"Color map to use for the heatmap"*: Blues # Or what you want
>        - *"The x-axis label"*: `distance from peak center (bp)`
>        - *"The y-axis label for the top panel"*: `CTCF peaks` # I don't know why this does not work currently...
>        - *"Reference point label"*: `peak center`
{: .hands_on}

> ### {% icon comment %} plotHeatmap Results
> This is what you get from plotHeatmap, this is much more symetric:
> ![plotHeatmap output on CTCF](../../images/atac-seq/plotHeatmapOutput_CTCF.png "plotHeatmap output on CTCF")
{: .comment}


## Visualise Regions with pyGenomeTracks

In order to visualise a specific region (e.g. the gene *RAC2*), we can either use a genome browser like **IGV** or **UCSC browser**, or use **pyGenomeTracks** to make publishable figures. We will use **pyGenomeTracks**.

> ### {% icon hands_on %} Hands-on: Task description
>
> 1. **pyGenomeTracks** {% icon tool %} with the following parameters:
>    - *"Region of the genome to limit the operation"*: `chr22:37,193,000-37,252,000`
>    - In *"Include tracks in your plot"*:
>        - *"1. Include tracks in your plot"*
>            - *"Choose style of the track"*: `Bigwig track `
>                - *"Plot title"*: `Coverage from MACS2 (extended +/-100bp)`
>                - {% icon param-file %} *"Track file bigwig format"*: Select the output of **Wig/BedGraph-to-bigWig** {% icon tool %} called `MACS2 bigwig`.
>                - *"Color of track"*: Select the color of your choice
>                - *"Minimum value"*: 0
>                - *"height"*: `5`
>                - *"Show visualization of data range"*: `Yes`
>        - {% icon param-repeat %} *"Insert Include tracks in your plot"*
>            - *"Choose style of the track"*: `NarrowPeak track`
>                - *"Plot title"*: `Peaks from MACS2 (extended +/-100bp)`
>                - {% icon param-file %} *"Track file bed format"*: Select the output of **MACS2** {% icon tool %} (narrow Peaks).
>                - *"Color of track"*: Select the color of your choice
>                - *"display to use"*: `box: Draw a box`
>                - *"Plot labels (name, p-val, q-val)"*: `No`
>        - {% icon param-repeat %} *"Insert Include tracks in your plot"*
>            - *"Choose style of the track"*: `Gene track / Bed track`
>                - *"Plot title"*: `Genes`
>                - {% icon param-file %} *"Track file bed format"*: `chr22 genes`
>                - *"Color of track"*: Select the color of your choice
>                - *"height"*: `5`
>                - *"Put all labels inside the plotted region"*: `Yes`
>                - *"Allow to put labels in the right margin"*: `Yes`
>        - {% icon param-repeat %} *"Insert Include tracks in your plot"*
>            - *"Choose style of the track"*: `NarrowPeak track`
>                - *"Plot title"*: `CTCF peaks`
>                - {% icon param-file %} *"Track file bed format"*: Select the first dataset: `ENCFF933NTR.bed.gz`
>                - *"Color of track"*: Select the color of your choice
>                - *"display to use"*: `box: Draw a box`
>                - *"Plot labels (name, p-val, q-val)"*: `No`
>        - {% icon param-repeat %} *"Insert Include tracks in your plot"*
>            - *"Choose style of the track"*: `X-axis`
>
> 2. Click on the {% icon galaxy-eye %} (eye) icon of the output.
>
{: .hands_on}

> ### {% icon comment %} pyGenomeTracks Results
> You should get similar to results to this from pyGenomeTracks:
> ![pyGenomeTracks output](../../images/atac-seq/pyGenomeTracksOutput.png "pyGenomeTracks output")
{: .comment}

> ### {% icon question %} Questions
> In the ATAC-Seq sample in this selected region we see four peaks detected by MACS2.
>
> 1. How many TSS are accessible in the sample in the displayed region?
> 2. How many CTCF binding loci are accessible?
> 3. Can you spot peaks with no TSS and no CTCF peak?
>
> > ### {% icon solution %} Solution
> >
> > 1. In total, we can see 3 TSS for 6 transcripts for 2 genes. The TSS of RAC2 corresponds to an ATAC-Seq peak whereas there is no significant coverage on both TSS of SSTR3.
> >
> > 2. Only the first peak on the left overlaps with a CTCF binding site.
> >
> > 3. Amongst the 4 peaks in this region, the 2 in the middle do not correspond to CTCF peaks or TSS.
> >
> {: .solution}
>
{: .question}

As CTCF creates accessible regions, a region containing a peak with no corresponding CTCF peak or TSS could be a putative enhancer. In the pyGenomeTracks plot we see a region like this located in the intron of a gene and another one between genes. However, it is impossible to guess from the position which would be the gene controlled by this region. And of course, more analyses are needed to assess if it is a real enhancer, for example, histone ChIP-seq, 3D structure, transgenic assay, etc.


# Conclusion

In this training you have learned the general principles of ATAC-Seq data analysis. ATAC-Seq
is a method to investigate the chromatin accessibility and the genome is treated with
a transposase (enzyme) called Tn5. It marks open chromatin regions by cutting and
inserting adapters for sequencing. The training material gave you an insight into how to quality control the data. You should look for low quality bases, adapter contamination, correct insert size and PCR duplicates (duplication level). We showed you how to remove adapters and PCR duplicates, if **FastQC**, shows a warning in these areas. We mapped the reads
with **Bowtie2**, filtered our reads for properly paired, good quality and reads that do not
map to the mitochondrial genome. We found open chromatin regions with **MACS2**, a tool to find regions of genomic enrichment (peaks). We investigated the read coverage around TSS with the help of **computeMatrix** and **plotHeatmap**. Last but not least, we visualised the peaks and other informative tracks, such as CTCF binding regions and hg38 genes, with the help of **pyGenomeTracks**. At the end, we found open chromatin regions that did not overlap with CTCF sites or TSS, which could be potential putative enhancer regions detected by the ATAC-Seq experiment.

![ATAC workflow](../../images/atac-seq/ATACWF.svg "ATAC workflow")
