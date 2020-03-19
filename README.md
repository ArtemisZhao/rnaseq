# RNAseq Pipeline

## Build the index for STAR and RSEM
 
### Generate genome STAR indexes

In this step, user should provide the reference genome sequences (FASTA) file and annotations (GTF) file, from which STAR generates genome indices that are utilized in the upcoming mapping/alignment step. The genome indexes are saved to disk and need only be generated once for each genome/annotation combination. 


```{r,eval=FALSE}
$PathToSTAR/STAR 
--runMode genomeGenerate 
--genomeDir $PathToIndex/star_index_oh75 
--genomeFastaFiles $PathToFasta/Homo_sapiens_assembly19.fasta 
--sjdbGTFfile $PathToGTF/gencode.v19.annotation.patched_contigs.gtf --sjdbOverhang 75
--runThreadN 4
```

Assume that the installed STAR is located at the current location. Replace $PathToSTAR/STAR with STAR/bin/Linux_x86_64/STAR. And fasta and gtf files are located under the path as described above.

The generated STAR indices are saved under the user defined path $PathToIndex/star_index_oh75.


### Build RSEM index

```{r, eval=FALSE}
$PathToRSEM/rsem-prepare-reference 
$PathToFasta/Homo_sapiens_assembly19.fasta
$PathToRef/rsem_reference/rsem_reference 
--gtf $PathToGTF/gencode.v19.annotation.patched_contigs.gtf 
--num-threads 4
```

The rsem-prepare-reference is a function in RSEM program and we assume that the installed RSEM is located under the directory $PathToRSEM. And fasta and gtf files are located under the path as described above (should be the same with the ones used in building STAR index).

The generated RSEM indices are saved under the user defined path $PathToRef/rsem_reference.

## Convert bam to fastq
Bam file is a compressed binary version of a SAM file, we convert it to a more organized format, which is FASTQ file, a text-based and storing both biological sequences and their corresponding quality score.

We will use the python files that built up by broadinstitute. Located in https://github.com/broadinstitute/gtex-pipeline/tree/master/rnaseq/src

```{r,eval=FALSE}
python3 $PathToPy/run_SamToFastq.py 
/data/$input.bam 
-p $outputname 
-o /data
```

We assume that the directory $PathToPy contains all the python files that are going to be used. And for simplicity, assume /data directory contains all the original files and will store all the generated files as well.

In this step, three fastq.gz files will be generated. Notice that they contain the user-defined name "outputname".

## STAR alignment
In this step, we align reads to the reference genome. 

```{r,eval=FALSE}
python3 $PathToPy/run_STAR.py 
$PathToIndex/star_index_oh75 
/data/outputname_1.fastq.gz 
/data/outputname_2.fastq.gz 
-threads 4 
--output_dir /data/star_out 
--chimOutType SaperateSAMold
```

The input contains, generated star index in the very beginning, which stores at $PathToIndex/star_index_oh75; and the two output fastq.gz files generated in the above step. 

We store the output files in the directory /data/star_out. 

## Mark duplicates
In this step, we locate and tag duplicate reads in a BAM file, where duplicate reads are defined as originating from a single fragment of DNA.

```{r,eval=FALSE}
python3 $PathToPY/run_MarkDuplicates.py
star_out/outputname.fastq.gz.Aligned.sortedByCoord.out.bam
outputname.Aligned.sortedByCoord.out.patched.md 
--output_dir data/ 
--jar $PathTopicard/picard.jar
```

The input file <outputname.fastq.gz.Aligned.sortedByCoord.out.bam> is generated in the above STAR alignment step, which is under data/star_out directory.

The file <outputname.Aligned.sortedByCoord.out.patched.md> is generated in the converting step, and it should be under the data directory.

Notice that $PathTopicard directs to the directory where picard installs.

## RNA-SeQC
In this quality control step, we compute three types of quality control metrics: read count with particular characteristics, coverage and expression correlation.

```{r,eval=FALSE}
$PathToRNASeQC/rnaseqc
$PathToGTF/gencode.v7.annotation_goodContig.gtf  data/outputname.Aligned.sortedByCoord.out.patched.md.bam 
data/ 
```

Assume the RNASeQC tools are located under $PathToRNASeQC.

The input file <outputname.Aligned.sortedByCoord.out.patched.md.bam> is generated from the mark duplicates step. 

All the output will be stored in the data/ directory.

## RSEM Transcription quantification
In this step, we map reads to transcriptome and calculate the transcript expression quantification.

```{r,eval=FALSE}
python3 $PathToPY/run_RSEM.py 
$PathToRef/rsem_reference
star_out/outputname.fastq.gz.Aligned.toTranscriptome.out.bam 
data/ 
--threads 4
```

The generated rsem reference files are installed under $PathToRef/rsem_reference as described in the above step.

The input data <outputname.fastq.gz.Aligned.toTranscriptome.out.bam> is generated in the star alignment step and stored under data/star_out directory. All the output will be generated in the data directory.


