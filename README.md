# Nextflow RNA-seq training

My work following the [Nextflow for Science: RNAseq](https://training.nextflow.io/latest/nf4_science/rnaseq/) training (`nf4-science/rnaseq`).

The method follows the bulk RNA-seq exercise from the [Babraham Institute](https://www.bioinformatics.babraham.ac.uk/training/RNASeq_Course/Analysing%20RNA-Seq%20data%20Exercise.pdf). Before writing any Nextflow code, each step is first tested manually, in containers, on a small test dataset (Part 1). The commands are then wrapped into a Nextflow workflow (Parts 2 and 3).

## Pipeline steps

1. **FastQC**: quality control of raw reads
2. **Trim Galore** (Cutadapt + FastQC): adapter trimming and post-trimming QC
3. **HISAT2** (+ samtools): spliced alignment to the reference genome
4. **MultiQC**: aggregated QC report across samples

## Dataset

Provided by the training (small region of human chromosome 20, hg19/b37):

- `reads/`: FASTQ files (single-end reads used here)
- `genome.fa`: reference genome
- `single-end.csv`, `paired-end.csv`: sample sheets

Samples processed manually (Part 1): `ENCSR000COQ1`, `ENCSR000COQ2`, `ENCSR000COR1`. The Part 2 workflow runs on `ENCSR000COQ1`.

The data is not included in this repository. To rerun the workflow, get the `nf4-science/rnaseq/data` folder from [nextflow-io/training](https://github.com/nextflow-io/training) and place it inside `workflow/` (next to `rnaseq.nf`). The genome index archive `data/genome_index.tar.gz` must also be built first, as shown in Part 1 (`hisat2-build` + `tar`).

## Part 1: Manual processing

All commands are run from the `nf4-science/rnaseq` directory, with `./data` mounted in each container.

### 1. Single sample (ENCSR000COQ1)

**QC and trimming** (FastQC + Trim Galore)

```bash
docker pull community.wave.seqera.io/library/trim-galore:0.6.10--1bf8ca4e1967cd18
docker run -it -v ./data:/data community.wave.seqera.io/library/trim-galore:0.6.10--1bf8ca4e1967cd18

# inside the container
fastqc /data/reads/ENCSR000COQ1_1.fastq.gz
trim_galore --fastqc /data/reads/ENCSR000COQ1_1.fastq.gz
mkdir /data/trimmed
mv ENCSR000COQ1_1* /data/trimmed
exit
```

**Alignment** (HISAT2 + samtools)

```bash
docker pull community.wave.seqera.io/library/hisat2_samtools:5e49f68a37dc010e
docker run -it -v ./data:/data community.wave.seqera.io/library/hisat2_samtools:5e49f68a37dc010e

# inside the container: build the genome index and archive it
hisat2-build /data/genome.fa genome_index
tar -czvf /data/genome_index.tar.gz genome_index.*

# align the trimmed reads and write a BAM file
hisat2 -x genome_index -U /data/trimmed/ENCSR000COQ1_1_trimmed.fq.gz \
    --new-summary --summary-file ENCSR000COQ1_1_trimmed.hisat2.log | \
    samtools view -bS -o ENCSR000COQ1_1_trimmed.bam

mkdir /data/aligned
mv ENCSR000COQ1_1* /data/aligned
exit
```

### 2. Additional samples and aggregated QC (ENCSR000COQ2, ENCSR000COR1)

**QC and trimming**

```bash
docker run -it -v ./data:/data community.wave.seqera.io/library/trim-galore:0.6.10--1bf8ca4e1967cd18

# inside the container
fastqc /data/reads/ENCSR000COQ2_1.fastq.gz
fastqc /data/reads/ENCSR000COR1_1.fastq.gz
trim_galore --fastqc /data/reads/ENCSR000COQ2_1.fastq.gz
trim_galore --fastqc /data/reads/ENCSR000COR1_1.fastq.gz
mv ENCSR000COQ2_1* ENCSR000COR1_1* /data/trimmed
exit
```

**Alignment**

```bash
docker run -it -v ./data:/data community.wave.seqera.io/library/hisat2_samtools:5e49f68a37dc010e

# inside the container: restore the genome index
tar -xzf /data/genome_index.tar.gz

hisat2 -x genome_index -U /data/trimmed/ENCSR000COQ2_1_trimmed.fq.gz \
    --new-summary --summary-file ENCSR000COQ2_1_trimmed.hisat2.log | \
    samtools view -bS -o ENCSR000COQ2_1_trimmed.bam

hisat2 -x genome_index -U /data/trimmed/ENCSR000COR1_1_trimmed.fq.gz \
    --new-summary --summary-file ENCSR000COR1_1_trimmed.hisat2.log | \
    samtools view -bS -o ENCSR000COR1_1_trimmed.bam

mv ENCSR000COQ2_1* ENCSR000COR1_1* /data/aligned
exit
```

**MultiQC report**

```bash
docker pull community.wave.seqera.io/library/pip_multiqc:a3c26f6199d64b7c
docker run -it -v ./data:/data community.wave.seqera.io/library/pip_multiqc:a3c26f6199d64b7c

# inside the container
multiqc /data/reads /data/trimmed /data/aligned -n all_samples_QC
mkdir /data/multiqc
mv all_samples_QC* /data/multiqc
exit
```

Outputs are in `results_manual/`.

## Part 2: Single-sample workflow

The manual commands are wrapped into a Nextflow workflow (FASTQC → TRIM_GALORE → HISAT2_ALIGN), with one module per tool and one container per process.

- `workflow/rnaseq.nf`: main workflow
- `workflow/modules/`: `fastqc.nf`, `trim_galore.nf`, `hisat2_align.nf`
- `workflow/nextflow.config`: configuration (Docker enabled, `test` profile with default input paths)

Inputs are set with two parameters, given default values by the `test` profile (paths relative to `workflow/`):

- `--input`: FASTQ file (`data/reads/ENCSR000COQ1_1.fastq.gz`)
- `--hisat2_index_zip`: HISAT2 genome index archive (`data/genome_index.tar.gz`, built in Part 1)

**Run** (from `workflow/`, with Docker running):

```bash
nextflow run rnaseq.nf -profile test
```

Outputs are published to `results/{fastqc,trimming,align}` as symbolic links to the `work/` directory. Light outputs from my run were copied as real files into `results_part2/`.

## Repository content

- `results_manual/`: Part 1 outputs
  - `aligned/`: HISAT2 alignment logs
  - `trimmed/`: trimming reports and post-trimming FastQC reports
  - `multiqc/`: aggregated MultiQC report (`all_samples_QC.html`) and its data
- `workflow/`: Nextflow workflow and modules (Part 2)
- `results_part2/`: Part 2 outputs (FastQC, trimming reports, HISAT2 log)

Large files (FASTQ, BAM, genome index) and Nextflow working files are excluded via `.gitignore`.

## Tools and containers

| Tool | Container |
|---|---|
| FastQC, Trim Galore 0.6.10 | `community.wave.seqera.io/library/trim-galore:0.6.10--1bf8ca4e1967cd18` |
| HISAT2, samtools | `community.wave.seqera.io/library/hisat2_samtools:5e49f68a37dc010e` |
| MultiQC | `community.wave.seqera.io/library/pip_multiqc:a3c26f6199d64b7c` |

Environment: WSL2 (Ubuntu), Java, Nextflow, Docker Desktop, VS Code.

## Progress

- [x] Part 1: manual processing of 3 samples
- [x] Part 2: single-sample Nextflow workflow
- [ ] Part 3: multi-sample Nextflow workflow