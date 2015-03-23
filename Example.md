# Example pipeline #

Here we provide a full example for going from RNA-Seq reads to gene-level differential expression results.
I'm going to use the yeast example running Trinity from our paper because it's small enough to run quickly,
but you can replace the file names with whatever data you have.

You can download the yeast data from [here](http://sra.dnanexus.com/runs/SRR453566). You will need to download files SRR453566 to SRR453571. SRA provides tools to covert from their format to others [here](http://trace.ncbi.nlm.nih.gov/Traces/sra/sra.cgi?view=software). Use:
```
fastq-dump --split-3 SRR453566.sra
```
on each file to get paired-end fastq files.

## Prepare the reads ##
It's worth while cleaning the reads before running the de novo assembler. I like
to use [trimmomatic](http://www.usadellab.org/cms/index.php?page=trimmomatic) to trim off any ends which have low quality scored. It's fast and can handle paired-end reads well.
```
trimmomatic PE -phred33 SRR453566_1.fastq SRR453566_2.fastq SRR453566_P1.fastq SRR453566_U1.fastq SRR453566_P2.fastq SRR453566_U2.fastq LEADING:20 TRAILING:20 MINLEN:50
```
P here means paired and U means unpaired. We will ignore the unpaired reads for the reminder of this tutorial.

## Perform the de novo assembly ##

In this example we will use [Trinity](http://trinityrnaseq.sourceforge.net/) to assemble the transcripts. You will have more power to detect transcripts if all the samples are pooled, therefore the first step is to concatenate the fastq files together:
```
cat *_P1.fastq | sed 's/ HWI/\/1 HWI/g' > all_1.fastq
cat *_P2.fastq | sed 's/ HWI/\/2 HWI/g' > all_2.fastq
```
The middle part of this code fixes the read IDs for Trinity by adding a \1 to the end of
the first end's name and a \2 to the end of the 2nd ends name. This is a hack that
I discovered is necessary for running Trinity on SRA data. Without fixing the read IDs,
Trinity will loose the paired-end information. For Illumina Hi-Seq reads, I've found that
Trinity works fine without doing this. You can check that the reads-ends are being interpreted correctly
by looking at read IDs in the `both.fa` file the Trinity creates. If something has gone wrong you'll
see an ID like ">SRR453566.19/H". Where the "H" should clearly be a "1" or "2".

now run Trinity:
```
<path to Trinity>/Trinity.pl --JM 10G --seqType fq --left all_1.fastq --right all_2.fastq
```

Note that this might take several hours to run. The result should be a fasta file of transcripts called `Trinity.fasta`

A couple of tricks in getting Trinity to work:
  * I have alway needed to set `ulimit -s unlimited`
  * I have often needed to increase the memory used by jellyfish. e.g. `--JM 100G`
  * I like to use the `--full_cleanup` option so I don't get hundres of big files left over.
  * Speed up the assembly by running in parallell. e.g. `--CPU 16`. However sometimes when you do this, you'll find that the job fails at the end because too many processes want memory. If this happens rerun Trinity with a smaller number of CPUs. It should restart where it stopped.


## Map reads back to the transcriptome ##

Now we need to multi-map the reads back to the transcriptome. In this example I will use [bowtie](http://bowtie-bio.sourceforge.net/index.shtml). The bowtie option `--all` means that all alignments are reported. If you have a large dataset, you might like to limit the number of reported
alignment to a large (but finite) number, as this will be faster and the resulting bam files will be smaller. In bowtie you could do this with `-k 40` (40 reported alignments).

In the paper, and this example, we map the reads back as pair-end, meaning that if only one read end maps to a transcript and not both, neither is reported. There may be some disadvantages in this, but we have not experimented with it. Feel free to try different mapping options, such as single-end and let us know if it makes a difference!

Build the bowtie index:
```
bowtie-build Trinity.fasta Trinity
```

Map each file to the transcriptome. Here is a short bash script for doing that:
```
FILES=`ls SRR*_P1.fastq | sed 's/_P1.fastq//g'`
for F in $FILES ; do
        R1=${F}_P1.fastq
        R2=${F}_P2.fastq
        bowtie --all -S Trinity -1 $R1 -2 $R2 > ${F}.sam  
        samtools view -S -b ${F}.sam > ${F}.bam
done
```
Speed this up by running bowtie with multiple threads eg. using the flag `-p 16`

You might want to delete the .sam files at this point if they take up too much space

## Run corset ##

The first three files (in numeric order) belong to one experimental group in our example while the final three belong to another. Batch versus chemostat growing conditions. We provide this information to Corset to improve the power it has when splitting differentially expressed paralogs (the -g option below gives these groupings). We also provide a name: A1..B3, for each of the size sample, which will appear in the header of the counts file.

```
corset -g 1,1,1,2,2,2 -n A1,A2,A3,B1,B2,B3 *.bam
```

## Run edgeR ##

[egdeR](http://www.bioconductor.org/packages/release/bioc/html/edgeR.html) is a bioconductor package in R which will be used to test the count data for significant differential expression. The snippet of
code below shows how you can get a list of the top differentially expressed clusters.

```
library(edgeR)

count_data<-read.delim("counts.txt",row.names=1)
group<-factor(c(1,1,1,2,2,2))

dge<-DGEList(counts=count_data,group=group)
dge<-calcNormFactors(dge)
dge<-estimateCommonDisp(dge)
dge<-estimateTagwiseDisp(dge)

results<-exactTest(dge)
topTags(results)
```


Clusters of interest can then be examined further and annotated. For example by selecting a representative transcript from each cluster and using software such as [blast2GO](http://www.blast2go.com/b2ghome) to work out what the gene is.

## Bpipe Pipeline ##

[bpipe](https://code.google.com/p/bpipe/) is a nice tool for managing pipelines and Simon Sadedin has proved an example of running the steps above through bpipe https://code.google.com/p/bpipe/wiki/RNASeqCorset


