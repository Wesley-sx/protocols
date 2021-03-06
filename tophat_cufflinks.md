# Analyzing RNA-seq data with the "Tuxedo" tools

_Introduction/tl;dr: I wrote this post as a reference for folks that are getting started with RNA-seq data analysis. It begins with an informal, big-picture overview of RNA-seq data analysis, and the rest of the post outlines one standard RNA-seq workflow (read alignment + transcriptome assembly, no differential expression analysis). Much of this protocol is specific to the JHSPH biostat computing setup._

### preliminaries: what's RNA-seq?
[RNA-seq](http://en.wikipedia.org/wiki/RNA-Seq) is a high-throughput technology used to measure gene expression in cell populations. For a super bare-bones picture of what gene expression is, please enjoy this ASCII art I made to illustrate the process:

```
[DNA]            ACGTAGGT{CGTATTT}AGCGT{AGCGCCCGA}TTACA
                                    |                   
                              transcription
                                    |
                                    V
[pre-RNA]                {GCAUAAA}UCGCA{UCGCGGGCU}
                                    |
                                 splicing
                                    |
                                    V
[RNA]                      {GCAUAAA}{UCGCGGGCU}                            
```

As shown in the awesome ASCII illustration, RNA molecules are basically strings of characters (G, C, U, or A; referred to as "nucleotides" or "bases"). RNA molecules are transcribed from DNA, and mature RNA consists only of the "expressed" parts of a gene (denoted in the illustration between curly brackets). Sequencing machines read off the base sequences of the mature RNA molecules in the cells in question. Base calling is kind of a hard problem, and current technologies can only read off about 100 nucleotides in a row, even though actual human RNA transcripts can be hundreds or thousands of bases long. These ~100-base reads are then written out into a text file in [FASTQ format](http://en.wikipedia.org/wiki/FASTQ_format). These are what I consider "raw" RNA-seq data.

### preliminaries II: why is RNA-seq important?
We can do lots of fun and scientifically-interesting things with RNA-seq data. I am mostly in the business of developing statistical methods to determine whether specific RNA transcripts are expressed at different levels in different cell populations (differential expression analysis). You can measure a transcript's expression level, or abundance in the cell, by figuring out how many RNA-seq reads came from that particular transcript. You can also do a lot of other cool stuff with RNA-seq, like discover new genes/transcripts, compare genes' expression levels to each other, or study the expression profile of a gene or transcript over time (e.g., which genes are expressed while a cell is developing from a stem cell into a heart cell?). Since there is so much we can learn from this type of data, it's being used to research complex diseases (e.g. cancer, psychiatric disease), evolutionary biology, organism development, and lots of other fields. If you'd like a more formal overview of RNA-seq, I highly recommend [this review paper](http://genomebiology.com/2010/11/12/220) (open access!). A more formal writeup of the analysis process is available in this [protocols paper](http://europepmc.org/articles/PMC3334321/) (also open access). 

### preliminaries III: what do I need to get started?

* A Linux cluster with the Sun Grid Engine scheduling system. (Definitely not required for RNA-seq analysis, but the workflow described here assumes it).
* [TopHat](http://ccb.jhu.edu/software/tophat/index.shtml) 
* [Cufflinks](http://cufflinks.cbcb.umd.edu/)
* A reference/index/set of annotations for the organism you're analyzing. I recommend downloading a reference from [this page](http://ccb.jhu.edu/software/tophat/igenomes.shtml). Any of the choices for your organism are fine. 
* A good text editor. I use [Sublime](http://www.sublimetext.com/) on my laptop, and I transfer files to the cluster using sftp (sometimes with the [Cyberduck](http://cyberduck.io/) GUI). To edit text files directly on the cluster, I use emacs and I open files for editing with `emacs -nw myfile.sh`. When I'm done editing the file, I close it with ctrl-x ctrl-c, then hit "y" to save.
* Data. This pipeline will work if you're starting with either FASTQ files (raw reads) or .bam files (read alignments). 

### step 0: setting up the project directory
I usually set up the directory for an RNA-seq analysis like this, if I'm starting with raw reads:
```
ProjectName/
├── data/
    ├── sample1_1.fastq
    ├── sample1_2.fastq
    ├── sample2_1.fastq
    ├── sample2_1.fastq
    ├── ...
    ├── sampleN_1.fastq
    └── sampleN_2.fastq
├── alignments/
    └── scripts/
├── assemblies/
    ├── scripts/
    └── merged/
├── DE_analysis/
├── reference/
    └── Homo_Sapiens/
```
(for transcript assembly with Cufflinks, usually we use paired-end reads, which is why there are 2 fastq files for each sample).

My setup will look like this, if I'm starting from read alignments (bam files):
```
ProjectName/
├── alignments/
    ├── sample1.bam
    ├── sample2.bam
    ├── ...
    └── sampleN.bam
├── assemblies/
    ├── scripts/
    └── merged/
├── DE_analysis/
├── reference/
    └── Homo_Sapiens/
```
You don't need new reference files for every experiment you analyze, but for this example, I'm just going to assume the reference is located in your project directory, so that the paths in the scripts make sense. 

### step 1: read alignment
**input**: RNA-seq reads, in FASTQ or FASTA format (extensions `.fastq`, `.fq`, `.fasta`, or `.fa`). They can be zipped/compressed.  
**output**: read alignments, in `.bam` format. (BAM is the compressed form of SAM, i.e. "sequence alignment/map" format, [described here](http://samtools.sourceforge.net/SAMv1.pdf)).  
**task**: determine where on the _genome_ (DNA) each RNA-seq read came from  
**run how many times:** once for each sample in your experiment  
**move to step 2 if:** you already have read alignments in .bam format

TopHat has to be run separately on every sample. For datasets with several samples, I like to run the TopHat jobs in parallel.  I also like to use multiple threads/cores to run each job, to improve speed; this is defined by the `-p` argument to TopHat. So for each sample, I submit a bash script that looks like this (defining some variables before the actual TopHat command so the command isn't totally unreadable):
```shell
#!/bin/sh

SAMPLE_ID=1
TOPHAT_BINARY=/path/to/tophat/executable/tophat
GENE_REFERENCE=/ProjectName/reference/Homo_sapiens/UCSC/hg19/Annotation/Genes/genes.gtf
BOWTIE_INDEX=/ProjectName/reference/Homo_sapiens/UCSC/hg19/Sequence/Bowtie2Index/genome
P=4 #use 4 threads

$TOPHAT_BINARY -G $GENE_REFERENCE -p $P -o /ProjectName/alignments/sample${SAMPLE_ID} $BOWTIE_INDEX /ProjectName/data/sample${SAMPLE_ID}_1.fastq /ProjectName/data/sample${SAMPLE_ID}_2.fastq
```

Note that there are [a LOT of TopHat parameters you can set](http://ccb.jhu.edu/software/tophat/manual.shtml). I generally use the defaults for most of them. Above, I specified `-G` and a path to a `genes.gtf` file, which means that I first want to align reads to the _transcriptome_ (i.e., known RNA), and _then_ map any remaining unmapped reads back to the _genome_. (One case where a read would map to the genome but not the transcriptome is if it came from a [retained intron](http://en.wikipedia.org/wiki/Alternative_splicing), so the sequence wouldn't appear in the annotated transcriptome). I also specified `-p` to tell TopHat to use multiple threads/cores (this is very helpful in terms of speed). The one parameter I didn't set above, but that I do want to mention, is the `-r` parameter, the mate inner distance -- if you have paired-end reads, you should set the mate inner distance to be the fragment length minus twice the read length (if you don't know what this means, talk to the person who gave you the data). The default is 50.

I don't really want to make a script by hand for every single sample, nor do I want to manually submit them all with `qsub`. There could be hundreds of samples in my experiment. To get around this, I create a master shell script, `tophat.sh`, in the main project directory. The master script creates the sample-specific scripts automatically, using the syntax `cat > filename.sh >> EOF`, and submits them with `qsub`. For the experiment outlined here, my `tophat.sh` file would look like this:

```shell
#!/bin/sh

TOPHAT_BINARY=/path/to/tophat/executable/tophat
GENE_REFERENCE=/ProjectName/reference/Homo_sapiens/UCSC/hg19/Annotation/Genes/genes.gtf
BOWTIE_INDEX=/ProjectName/reference/Homo_sapiens/UCSC/hg19/Sequence/Bowtie2Index/genome
P=4 

for SAMPLE_ID in {1..N}
do
cat > /ProjectName/alignments/scripts/tophat_${SAMPLE_ID}.sh <<EOF
$TOPHAT_BINARY -G $GENE_REFERENCE -p $P -o /ProjectName/alignments/sample${SAMPLE_ID} $BOWTIE_INDEX /ProjectName/data/sample${SAMPLE_ID}_1.fastq /ProjectName/data/sample${SAMPLE_ID}_2.fastq
mv /ProjectName/alignments/sample${SAMPLE_ID}/accepted_hits.bam /ProjectName/alignments/sample${SAMPLE_ID}.bam
EOF

qsub -l mf=20G,h_vmem=5G -pe local $P -m e -M myemail@email.com /ProjectName/alignments/scripts/tophat_${SAMPLE_ID}.sh

done
```

Then, from the `ProjectName` directory, just execute `tophat.sh`:
```
[afrazee@enigma2 ProjectName]$ sh tophat.sh
Your job tophat_sample1.sh has been submitted
Your job tophat_sample2.sh has been submitted
...
Your job tophat_sampleN.sh has been submitted 
```

You'll get an email at `myemail@email.com` when each job finishes. They'll take a few hours each.

If you look in the `scripts` subdirectory of the `alignments` folder, you can see the sample-specific scripts that the master script created. When your jobs are finished, your project directory will look like this (note that the final alignments end up in the `alignments` directory due to the `mv` command at the end of each sample-specific script):

```
ProjectName/
├── tophat.sh
├── data/
    ├── sample1_1.fastq
    ├── sample1_2.fastq
    ├── sample2_1.fastq
    ├── sample2_1.fastq
    ├── ...
    ├── sampleN_1.fastq
    └── sampleN_2.fastq
├── alignments/
    ├── sample1.bam
    ├── sample2.bam
    ├── ...
    ├── sampleN.bam
    ├── sample1/
        └── [...other TopHat output...]
    ├── sample2/
        └── [...other TopHat output...]
    ├── ...
        └── [...other TopHat output...]
    ├── sampleN/
        └── [...other TopHat output...]
    └── scripts/
        ├── tophat_sample1.sh
        ├── tophat_sample1.sh
        ├── ...
        └── tophat_sampleN.sh
├── assemblies/
    ├── scripts/
    └── merged/
├── DE_analysis/
├── reference/
    └── Homo_Sapiens/
```


### step 2: transcript assembly
**input**: read alignments, in `.bam` format  
**output**: a transcript assembly, in [gtf format](http://www.ensembl.org/info/website/upload/gff.html). (A GTF file is a plain-text file that specifies locations, strands, and structures of genomic features. Usually there is one row per exon/coding sequence, and the rightmost column specifies which exons belong to which transcripts/genes).  
**task**: reconstruct the set of RNA transcripts generated by each gene in each sample, based on the RNA-seq reads  
**run how many times:** once for each sample in your experiment  

After you align RNA-seq reads back to the genome, you are ready to reconstruct the transcripts present in your experiment based on those alignments using Cufflinks.  We need to assemble the transcriptomes for each sample separately. The assemblies will be merged (in step 3) to create an overall transcriptome assembly for the experiment. Here, I'm assuming you want to do a totally _de novo_ assembly (i.e., reconstruct the transcripts without using annotation to help). For instructions on doing a guided assembly, and to see the myriad of other options available, [check out the manual](http://cufflinks.cbcb.umd.edu/manual.html). 

I use the same type of script structure for Cufflinks as I do for TopHat (see previous step): write a master script, `cufflinks.sh`, in the main project directory, which will create and submits sample-specific scripts.

My master `cufflinks.sh` script would look like this:

```shell
#!/bin/sh

CUFFLINKS_BINARY=/path/to/cufflinks/binary/cufflinks
P=4 #use 4 threads/cores

for SAMPLE_ID in {1..N}
do
cat > /ProjectName/assemblies/scripts/cufflinks_${SAMPLE_ID}.sh <<EOF 
#!/bin/sh

$CUFFLINKS_BINARY -q -p $P -o /ProjectName/assemblies/sample${SAMPLE_ID} /ProjectName/alignments/sample${SAMPLE_ID}.bam

mv /ProjectName/assemblies/sample${SAMPLE_ID}/transcripts.gtf /ProjectName/assemblies/sample${SAMPLE_ID}_transcripts.gtf
EOF
qsub -l mf=20G,h_vmem=5G -m e -M myemail@email.com -pe local $P cufflinks_${SAMPLE_ID}.sh
done
```

The `-q` option specifies that you want "quiet" output (the non-quiet output is very, very dense), and the `-p` option again specifies how many cores/threads to use. In my experience, multithreading gives you more of a speedup when you use it with TopHat than with Cufflinks (alignment is more easily parallelized), but it's still worth running Cufflinks on more than 1 core if you can.

And then from the `ProjectName` directory, I run:

```
[afrazee@enigma2 ProjectName]$ sh cufflinks.sh
Your job cufflinks_sample1.sh has been submitted
Your job cufflinks_sample2.sh has been submitted
...
Your job cufflinks_sampleN.sh has been submitted 
```

Again, you'll get an email notification when each sample's assembly is finished. When all the jobs are done, your project directory will now look something like this:
```
ProjectName/
├── tophat.sh
├── cufflinks.sh
├── data/
    ├── [...all data...]
├── alignments/
    ├── sample1.bam
    ├── sample2.bam
    ├── ...
    ├── sampleN.bam
    ├── [...all other TopHat output...]
    └── scripts/
        ├── [...all sample-specific TopHat scripts...]
├── assemblies/
    ├── sample1_transcripts.gtf
    ├── sample2_transcripts.gtf
    ├── ...
    ├── sampleN_transcripts.gtf
    ├── sample1/
        └── [...other Cufflinks output...]
    ├── sample2/
        └── [...other Cufflinks output...]
    ├── ...
        └── [...other Cufflinks output...]
    ├── sampleN/
        └── [...other Cufflinks output...]
    ├── scripts/
        ├── cufflinks_sample1.sh
        ├── cufflinks_sample2.sh
        ├── ...
        └── cufflinks_sampleN.sh
    └── merged/
├── DE_analysis/
├── reference/
    └── Homo_Sapiens/
```

And the assembly step is done!

### step 3: merge the sample-specific assemblies
**input**: the sample-specific assemblies, in `.gtf` format  
**output**: one merged assembly for the entire experiment, in `.gtf` format.  
**task**: intelligently combine the sample-specific assemblies into one main transcriptome, which will represent our estimate of the transcript structure present in this particular experiment. Statistical analysis will be performed on the merged assembly.  
**run how many times:** once for the entire project  

Compared to the other steps, this one is pretty short!

#### make a text file containing paths to all the sample-specific assemblies
I always call this file `assemblies.txt` and stick it in the `assemblies` subdirectory. For this project, the `assemblies.txt` file would look like this:

```
/ProjectName/assemblies/sample1_transcripts.gtf
/ProjectName/assemblies/sample2_transcripts.gtf
...
/ProjectName/assemblies/sampleN_transcripts.gtf
```
List each file out explicitly - the ... is just for illustrative purposes here :)

#### submit a cuffmerge script

Once you make `assemblies.txt`, you just need to run `cuffmerge` on that assembly list, with a few other parameters. You'll use a file from your reference. Here is my `cuffmerge.sh` script, which I put in the main project directory:

```shell
#!/bin/sh
#$ -cwd -l mf=20G,h_vmem=5G -pe local 4 -m e -M myemail@email.com

REFERENCE_SEQ=/ProjectName/reference/Homo_sapiens/UCSC/hg19/Sequence/Bowtie2Index/genome.fa
CUFFMERGE_BINARY=/path/to/cuffmerge/binary/cuffmerge
P=4

$CUFFMERGE_BINARY -s $REFERENCE_SEQ -p $P -o /ProjectName/assemblies/merged /ProjectName/assemblies/assemblies.txt
```

Then I just do:
```shell
[afrazee@enigma2 ProjectName]$ qsub cuffmerge.sh
Your job "cuffmerge.sh" has been submitted
```

A fun trick: when you have a line in a bash script prefixed with `#$`, qsub will parse all its arguments from there. So, the second line in the `cuffmerge.sh` script takes care of all my memory requests and whatnot, so I don't have to worry about it when I actually run my `qsub` command.

The output, `merged.gtf`, will be written to the `merged` folder specified by `-o`. Again, there are several other options, including one to do a guided merge (merge the assembled transcripts with annotated transcripts), detailed in the [manual](http://cufflinks.cbcb.umd.edu/manual.html) -- but this is my general template.

The project directory now looks like:
```
ProjectName/
├── tophat.sh
├── cufflinks.sh
├── cuffmerge.sh
├── data/
    ├── [...all data...]
├── alignments/
    ├── sample1.bam
    ├── sample2.bam
    ├── ...
    ├── sampleN.bam
    ├── [...all other TopHat output...]
    └── scripts/
        ├── [...all sample-specific TopHat scripts...]
├── assemblies/
    ├── sample1_transcripts.gtf
    ├── sample2_transcripts.gtf
    ├── ...
    ├── sampleN_transcripts.gtf
    ├── [...all other Cufflinks output...]
    ├── scripts/
        ├── [...all sample-specific Cufflinks scripts...]
    └── merged/
        ├── merged.gtf
        └── logs/
├── DE_analysis/
├── reference/
    └── Homo_Sapiens/
```

### step 4: profit

Congratulations! You now have a transcriptome assembly (`merged.gtf`) for your experiment, so you can do differential expression analysis (I mean, we already made a `DE_analysis` folder...it's just sitting there, waiting to be filled with statistics!). Or you can visualize your assembly. Or you can discover new isoforms. Or any number of other awesome things! The world of transcript analysis is your oyster. Have fun :) 

