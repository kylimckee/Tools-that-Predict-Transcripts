# Tools-that-Predict-Transcripts - TIdeS

Pipeline used to analyze rhabdomyosarcoma bulk RNA-sequencing data to identify 
novel open reading frames (ORFs) with the potential to encode microproteins.

---

### Make Microproteins Sequencing Project Working Directory

The anlysis requires a working directory where the raw FASTQ files can be accessed. 

```bash
mkdir microproteins_sequencing_project
cd /data/mckeeka/microproteins_sequencing_project
```

### Make Reference Working Directory and Add Human Reference Files

The anlysis requires a reference working directory where the human reference files can be accessed. 

```bash
mkdir reference
cd /data/mckeeka/microproteins_sequencing_project/reference

#Download GENCODE v50 human protein translations and UniProt human reference proteome
wget https://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_50/gencode.v50.pc_translations.fa.gz
wget https://ftp.uniprot.org/pub/databases/uniprot/current_release/knowledgebase/reference_proteomes/Eukaryota/UP000005640/UP000005640_9606.fasta.gz
gunzip *.gz

#Combine the two protein FASTA files
cat gencode.v50.pc_translations.fa \
    UP000005640_9606.fasta \
    > tides_human_combined.fa

#Remove 100% identical duplicates from thw GENCODE and UniProt files
cd-hit \
    -i tides_human_combined.fa \
    -o tides_human_reference.unique.fa \
    -c 1.00 \
    -n 5

#Convert protein FASTA into a DIAMOND database
diamond makedb \
    --in tides_human_reference.nr.fa \
    --db tides_human_reference
```

### Make TIdeS Working Directory 

```bash
mkdir TIdeS
cd /data/mckeeka/microproteins_sequencing_project/TIdeS
```

## Generate TIdeS Pipeline Configuration

This pipeline was generated to perform TIdeS analysis of the transcript FASTA files.

### Install TIdeS Tools

```bash
cd /data/mckeeka/microproteins_sequencing_project
conda create -n TIdeS -c bioconda snakemake tides-ml diamond cd-hit barrnap kraken2 -y
conda activate TIdeS
```

### Create Snakemake Raw QC Configuration File

```bash
nano TIdeS_pipeline.smk

# Add the following code to the configuration file:

SAMPLES = glob_wildcards("MCI_fastq_117_STS_FASTQ/{sample}.R1.fastq.gz").sample
READS = ["R1", "R2"]

TIDES_DB = "/data/mckeeka/microproteins_sequencing_project/reference/tides_human_reference.dmnd"

rule all:
    input:
        expand(
            "TIdeS/{sample}/TIdeS_complete.txt",
            sample=SAMPLES
        )

rule tides:
  input:
      transcripts="transcripts/{sample}_transcripts.fa",
      db=TIDES_DB
  output:
      done="TIdeS/{sample}/TIdeS_complete.txt"
  threads: 16
  params:
      outdir="TIdeS/{sample}",
      prefix="{sample}",
      genetic_code=1
  shell:
      """
      mkdir -p {params.outdir}

      cd {params.outdir}

      tides \
          -i ../../../{input.transcripts} \
          -o {params.prefix} \
          -d {input.db} \
          -t {threads} \
          -l 150 \
          -g {params.genetic_code} \
          -p

      touch TIdeS_complete.txt
      """
```

### Run Raw QC Configuration File

The pipeline must be run using sbatch on the Biowulf cluster.

```bash
cd /data/mckeeka/microproteins_sequencing_project
sbatch --cpus-per-task=1 --time=03-00:00:00 --wrap "snakemake -s TIdeS_pipeline.smk --jobs 16"
```
