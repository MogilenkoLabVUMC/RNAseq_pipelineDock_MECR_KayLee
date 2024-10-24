# General information

This is the fork, representing state of the Docker bulk RNAseq processing [pipeline](https://github.com/tony-zhelonkin/RNAseq_pipelineDock) at a timepoint, when it was used for pre-processing the data generated from experiments in the [paper](https://www.biorxiv.org/content/10.1101/2024.07.08.602554v1) *`Mitochondrial Fatty Acid Synthesis and Mecr Regulate CD4+ T Cell Function and Oxidative Metabolism`, 2024* by KayLee Steiner, Jeffrey Rathmell et al.

Data processing and downstream analysis done by [Anton Zhelonkin](https://github.com/tony-zhelonkin), supervised by [Denis Mogilenko](https://github.com/MogilenkoLab)

The code for downstream analysis is available [here](https://github.com/MogilenkoLabVUMC/MECR_KO_in_vivo-KayLee)

# The Pipeline logic

Below is the general logic of the pipeline.

![image](https://github.com/MogilenkoLabVUMC/RNAseq_pipelineDock_MECR_KayLee/blob/master/RnaSeqDock.png?raw=true)

*Note!* The pipeline is just a set of separate scripts currently. You need to tailor the scripts, the paths to your use case and the data at hand.
It`s not a press-one-button story for the moment.

# Processing details

The exact processing vignette for the dataset was as follows.

## 1. Build the Docker Image

Build the docker with:

```
docker build -t <my_image_name> base
```

In case of errors it`s usefull to build an image from scratch without using the cache from interim builds logging all the stderr and stdout:

```
docker build --no-cache -t <my_image_name> . > >(tee -a log.txt) 2> >(tee -a log.txt >&2)
```


## 2. Load into the interactive Docker shell under tmux session

First, I create a tmux session. I name the session with project I`m working on

```
tmux new -s <session_name>
```

Then I create a docker container based off of the Docker image, I built earlier. I mount all the necessary folders, that I`ll need to have for my work

```
docker run --name <docker_container_name> \
    -v /path/to/Docker/base/scripts:/scripts \
    -v /path/to/data/folder/with/fasta_reads:/data/Mecr \
    -v /path/to/reference_genome:/reference_genome \
    -it <docker_image_name> bash
```

Once you run the command you end up in the interactive `bash` shell under the Docker container environment. Keep in mind that all the paths will be relative to the Docker virtual environment.

## 3. Build the reference genome index

Build index with sjdbOverhang 100, which performs well for reads of various length

```
STAR --runMode genomeGenerate \
	 --runThreadN 8 \
     --genomeDir /genome_index \
     --genomeFastaFiles /path/to/reference_genome/GCF_000001635.27_GRCm39_genomic.fna \
     --sjdbGTFfile /path/to/reference_gtf_annotation/GCF_000001635.27_GRCm39_genomic.gtf \
     --sjdbOverhang 100
```

For the reference genome [GRCm39](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_000001635.27/) (mm39, GCF_000001635.27) was used.
Additional annotations, some of which are converted gtf-format annotations, required for some of the processing tools can be found in the *`annotations`* folder.

## 4. Pre-alignment QC

To inspect the QC metric before the alignment we used *`getPreAlignmentQC.sh`* script

```
/path/to/script/getPreAlignmentQC.sh
```

We trimmed the reads off of the Illumina adapter sequences and filtered for read quality using *`Trimmomatic`* under the *`runTrim.sh`* script. The *`Trimmomatic`* parameters were as follows:

```
trimmomatic PE -threads 8 \
  "${R1_FILE}" "${R2_FILE}" \
  "${TRIMMED_R1}" /dev/null \
  "${TRIMMED_R2}" /dev/null \
  ILLUMINACLIP:"${ADAPTERS}":"${ILLUMINACLIP_SETTINGS}" \
  LEADING:3 TRAILING:3 \
  SLIDINGWINDOW:4:20 MINLEN:36 \
```
The adapters folder in the Docker built is located in *`ADAPTERS="/usr/share/Trimmomatic-0.39/adapters/TruSeq3-PE.fa` and the ILLUMINACLIP_SETTINGS were "2:30:10".

## 5. Align the reads

To align the reads to the reference genome we used *`runSTARalign.sh`* script. STAR was run with the parameters:

```
STAR --runThreadN 10 \
             --genomeDir \"$GENOME_DIR\" \
             --readFilesIn \"$R1_FILE\" \"$R2_FILE\" \
             --readFilesCommand zcat \
             --sjdbOverhang 100 \
             --limitBAMsortRAM 45000000000 \
             --outFileNamePrefix \"$OUTPUT_DIR/${BASE_NAME}_\" \
             --outSAMtype BAM SortedByCoordinate \
             --outSAMattrRGline ID:$ID LB:$ID SM:$SM PL:$PL PU:$PU \
             --twopassMode Basic
```

## 6. Post-alignment QC

* *`getPostAlignmentQC.sh`* script is used to get various alignment QC metrics
* *`dedupPicard_postQC.sh`* script is used to de-duplicate the reads

For all the Picard QC tools we specified STRAND_SPECIFICITY="SECOND_READ_TRANSCRIPTION_STRAND", as the reads came from [Illumina Stranded mRNA](https://knowledge.illumina.com/library-preparation/rna-library-prep/library-preparation-rna-library-prep-reference_material-list/000002238) protocol.
Further down the analysis pipeline we investigated the read counts, that we got from trimmed reads against the one`s that we got from: trimmed reads -> Picard-deduplicated .bam files. Picard-deduplication introduced >50% ties in at the gene set enrichment analysis step, and so all the analysis downstream was performed for un-deduplicated data.

## 7. Calculate read counts matrix

For the calculation of the read count matrix we used *`featureCounts`* from under the script *`runFeatureCounts_reverse_stranded.sh`*. The script exrtacts raw RNAseq counts and order sample columns by sample name lexicographically.

The `featureCounts` parameters were the following:

```
featureCounts \
    -a "$ANNOTATION_FILE" \
    -o "$COUNTS_FILE" \
    -p \
    --countReadPairs \
    -B \
    -C \
    -s 2 \
    -t exon \
    -T "$THREADS"
```

* **`-p`** and **`--countReadPairs`** for paired-end mode, counting read pairs
* **`-B`** to calculate only reads that have both ends successfully
aligned
* **` -C`** to NOT count the chimeric fragments (those fragments that have
their two ends aligned to different chromosomes)
* **`-s 2`** for reversely stranded experiment
* **`-t exon`** to count exons (as that is what the final spliced transcript is composed of)
We haven\`t specified any parameters on multimapper calculation (**`-M`**), thus omitting multimapper summarization.
