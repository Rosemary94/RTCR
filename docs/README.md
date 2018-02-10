RTCR
====

Recover T Cell Receptor (RTCR) is a pipeline for complete and accurate
recovery of TCR repertoires from high throughput sequencing data.

Installation
============

RTCR requires an external aligner, we have chosen for Bowtie 2, but if
you would like to use your own, read the section "switching aligners".
The pipeline has been successfully tested with python version 2.7.3.

1.  Install [Bowtie
    2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml)
2.  Install the [RTCR package](https://uubram.github.io/RTCR)

    Download the zip file and extract it. Bowtie 2 is not included in
    this file and needs to be downloaded and installed separately (see
    step 1). Open a terminal and go to the newly extracted RTCR folder
    and run:

        python setup.py install

    To test if everything went well, type "rtcr" (and press enter). This
    should result in a help being printed showing the command line
    options of RTCR.

3.  Configure RTCR

    RTCR needs to know where Bowtie 2 is installed, specifically it
    needs to know the directory where the 'bowtie2-build' and 'bowtie2'
    executables reside:

        rtcr Config Aligner.location=/path/to/bowtie2/

    For more details on configuring RTCR, see the "Configuration"
    section

Known issues
============

1.  If there are permission issues during installation (e.g. not having
    root access) run:

        python setup.py install --user

2.  If the vtrie module fails to install during installation, first run:

        pip install vtrie

3.  If there is a lack of memory this may result in the pipeline exiting during
    error correction with a "Error occurred in worker pool" message. On Linux,
    this error can be the result of the Out of Memory (OOM) killer taking out a
    child process of RTCR (the loss of a child process will be reported by the
    pipeline in "rtcr.log").

Quick start
===========

Analysing the non-barcoded sample:

    mkdir my_analysis
    cd my_analysis
    rtcr run -i ../reads.fq.gz

Analysing the barcoded sample:

    mkdir my_barcoded_analysis
    cd my_barcoded_analysis
    rtcr Checkout -rc -b ../barcodes.txt -i ../umi_reads.fq.gz
    rtcr umi_group_ec -i S1.fastq -o S1_umi_group_ec.fastq
    rtcr run -i S1_umi_group_ec.fastq

Usage
=====

RTCR has a single entry-point, the "rtcr" command, with which everything
is run. With different arguments, RTCR can be configured, instructed to
do barcode error correction, or run an analysis.

RTCR takes FastaQ files as input. These files can be zipped to save disk
space. As an example, let us assume you have a zipped dataset called
"reads.fastq.gz" in a directory "/data", and the reads are from a human
TcR-beta repertoire.

First, create an empty directory, for example:

    cd /data
    mkdir my_analysis
    cd my_analysis

Now we can analyse the dataset as follows:

    rtcr run --reads ../reads.fastq.gz

RTCR will produce several files (see "Output files" section for more
details), but the retrieved clones will end up in a tab-delimited file
called "results.tsv". If something went wrong, please check the
"rtcr.log" file in the same directory.

> **important**
>
> Start analyses in an empty folder. If the alignments file
> "alignments.sam.gz" already exists, RTCR will *not* perform alignments
> and use this file instead.

> **note**
>
> If you have a paired-end HTS dataset, please merge the read-pairs
> first using a program such as
> [pear](http://sco.h-its.org/exelixis/web/software/pear/).

To use a different germline reference, for example to analyse a mouse
TcR-alpha repertoire HTS dataset (e.g. "mouse.fastq"), use the
"--species" and "--gene" options as follows:

    rtcr run --reads mouse.fastq --species MusMusculus --gene TRA

Finally, if you want to prevent RTCR from merging clones with the same
CDR3 but different VJ combination, use the "--no\_VJ\_collapsing"
option.

Output files
============

The "rtcr run" command produces many files, each of these will be
explained below:

-   

    alignments.sam.gz

    :   SAM records from the aligner, with alignments between the
        germline V and J sequences and the reads. The VJ alignments to
        the reads occur together, starting with the V alignments. Hence,
        the first record is a V alignment, the second a J alignment to
        the same read.

-   

    aln\_stats.csv

    :   Alignments statistics for the different sequence lengths (CDR3
        by default; "seqlen" column), counting the number of bases
        ("n"), mismatches ("mm"), insertions ("ins"), and deletions
        ("dels") encountered in the alignments of the V or J
        ("region" column). This data is used to calculate the various
        error rates that RTCR uses.

-   

    Q\_mm\_stats.csv

    :   For the V and J region alignments ("region" column), the number
        of bases ("n"), and number of mismatches ("mm") found of the
        bases having a certain Phred score ("Q"). This data is used by
        RTCR to calculate a normalization factor.

-   

    Q\_mm\_stats\_plot.pdf

    :   Graphical representation of Q\_mm\_stats.csv. This file is not
        created if matplotlib is not installed.

-   

    r.dat

    :   Raw clones identified by RTCR before error correction.

-   

    rqi.dat

    :   Clones after QMerge and IMerge algorithms have run.

-   

    rqil.dat

    :   Clones after QMerge, IMerge, and LMerge algorithms have run.

-   

    rqiln.dat

    :   Clones after QMerge, IMerge, LMerge, and NMerge algorithms
        have run.

-   

    results.tsv

    :   Final list of clones after error correction and post-processing.

> **note**
>
> Clones produced during the run are output to .dat files (see above).
> These can be converted to the same format as results.tsv using the
> Convert option. For example:
>
>     rtcr Convert -i r.dat -o r.tsv

Analysing a barcoded HTS dataset
================================

First RTCR needs to identify the unique molecular identifiers (UMIs) in
the reads. For this it requires the sequence of the primer(s) that
contain the UMI. Let's assume we have a barcoded HTS dataset
("umi\_reads.fastq") with one sample and a 12bp long UMI.

For this create a file call "barcodes.txt" with the following contents:

    S1      aagcagtggtaTCAACGcagagNNNNNNNNNNNNcttggggg

In the first column is the name of the sample, here "S1". The second
column, separated from the first by a tab, contains the primer sequence
where the "N" bases denote the location of the UMI. To look for the UMI,
RTCR will search the read for a 'seed' sequence, that is indicated in
the primer by a stretch of upper case bases (non-N). In the above it is
"TCAACG". This seed sequence is required as RTCR will search the read
for a perfect match to this sequence, and then search for the remaining
lower-case DNA bases. By default there are only 2 mismatches allowed in
the lower-case bases. To ask RTCR to search for the UMIs, run:

    rtcr Checkout -f reads.fastq -b barcodes.txt -rc

The "-rc" switch is used to tell RTCR to also look for the UMI in the
reverse complement of the reads. The above should create a file called
"S1.fastq". This file contains the reads in which RTCR managed to
identify a UMI.

Next, to perform barcode error correction:

    rtcr umi_group_ec -i S1.fastq -o S1_umi_group_ec.fastq

The "S1\_umi\_group\_ec.fastq" file contains the barcode error corrected
reads. After this one can perform the regular HTS analysis the same as
for non-barcoded HTS datasets:

    rtcr run --reads S1_umi_group_ec.fastq

Analysing a paired-end barcoded HTS dataset
===========================================

We here assume the raw reads have been merged using a program such as
[pear](http://sco.h-its.org/exelixis/web/software/pear/). In the latter
case, we can expect to have the following files:

    reads.assembled.fastq
    reads.unassembled.forward.fastq
    reads.unassembled.reverse.fastq
    reads.discarded.fastq

If almost all reads were successfully assembled, it is possible to
continue with only the reads.assembled.fastq file:

    rtcr Checkout -p reads.assembled.fastq -rc -b barcode.txt

However, if many reads were not assembled, then it is possible to take
along the unassembled reads as follows:

    rtcr Checkout -f reads.unassembled.forward.fastq -r reads.unassembled.reverse.fastq -rc -b barcode.txt

The output of the above command depends on barcode.txt, which contains
the barcode(s) rtcr should look for in the reads and sample names. If
there is one sample called "S1", then the above Checkout commands
produce the following three files (in order of the commands):

    S1_R12.fastq
    S1_R1.fastq
    S1_R2.fastq

These files contain all the read pairs (assembled (R12) and unassembled
(R1 and R2)) in which rtcr was able to find a barcode.

> **note**
>
> Names of reads in the above fastq files will have the UMI appended to
> the name, "UMI:XXX:YYY:ZZZ", where XXX is the read source (e.g. "R1"),
> and YYY and ZZZ are the UMI nt sequence and ascii encoded (Phred+33)
> base quality scores.

Next, use umi\_group\_ec for barcode error correction:

    cat S1_R1.fastq S1_R2.fastq S1_R12.fastq | rtcr umi_group_ec -o S1_umi_group_ec.fastq

> **note**
>
> 1.  
>
>     Currently, there is no check if a read-pair contains a CDR3 in both
>
>     :   R1 and R2. Therefore, it is technically possible for an
>         unassembled read to provide a CDR3 twice (though then the
>         question is why assembly failed).
>
> 2.  
>
>     The format of the read names in S1\_umi\_group\_ec.fastq is
>
>     :   "UMI:XXX:YYY:ZZZ:WWW", where XXX is the source e.g. "R1", YYY
>         is the UMI nt sequence, ZZZ is a fraction where the numerator
>         indicates the UMI group number and the denominator the number
>         of UMI groups that share the same UMI, and WWW is the number
>         of reads in the current UMI group.
>
Finally, rtcr can be run on the barcode corrected reads:

    rtcr run --reads S1_umi_group_ec.fastq

Configuration
=============

RTCR can be easily configured using the "rtcr Config" command. To find
out its usage, type:

    rtcr Config -h

To see the entire configuration file (can be big), type:

    rtcr Config

If you'd rather edit the config file directly, search for the
"config.ini" file in the "RTCR" folder of the package.

The ini file contains different sections, denoted with the brackets "\["
and "\]". These sections contain the different settings of RTCR. To
check the value of a setting, say for example the default germline
reference gene, type the name of its corresponding section and name of
the key (in this case it is the key "gene" in section "Defaults"):

    rtcr Config Defaults.gene

The above should print out "TRB". Let's for example change this default
to TCR alpha chains:

    rtcr Config Defaults.gene=TRA

Switching aligners
==================

To run rtcr with a different aligner, the values in the "\[Aligner\]"
section of its configuration file (see "Configuration") should be
updated. There are several requirements for the new aligner: 1) It
should support receiving FastaQ records via stdin (standard in) 2) It
should support writing SAM file format output to stdout (standard out)

If the aligner can do those things, then tell RTCR where the aligner is
located with the "location" key. If the aligner does not build an index,
empty out the corresponding keys as follows:

    rtcr Config Aligner.cmd_build_index=
    rtcr Config Aligner.args_build_index=

The "args\_XXX" keys are arguments that RTCR passes to commands given in
the "cmd\_XXX" keys. Before alignment, RTCR produces a reference fasta
file from the germline reference and passes this (using the
"args\_build\_index" key) to the command to build an index
("cmd\_build\_index" key). The "%(ref\_fn)s" and "%(index\_fn)s" in the
"args\_build\_index" key refer to the reference fasta file and index
filenames that RTCR uses internally.

RTCR first aligns the V genes to the reads and then the J genes. It is
possible to run the aligner with different arguments for both with the
"args\_align\_v" and "args\_align\_j" keys respectively. Any arguments
that are the same for both can be put in "args\_align\_base". The
"%phred\_encoding)s" and "%(n\_threads)s" parts of the arguments will
contain the phred encoding (33 or 64) and number of threads
respectively. It is optional to use these in the arguments.

Analyzing RTCR output with tcR R package
========================================

Data analysis of immune receptor repertoires can be performed using
[tcR](https://cran.r-project.org/web/packages/tcR/index.html), a package
for the [R](https://www.r-project.org/) software environment. RTCR
provides an R script, named 'tcR\_RTCR\_parser.R', for loading RTCR
output into the tcR package for analysis. See below for an example of
how to load RTCR output from inside R:

    source("tcR_RTCR_parser.R")
    rtcr_data <- parse.rtcr("results.tsv")
